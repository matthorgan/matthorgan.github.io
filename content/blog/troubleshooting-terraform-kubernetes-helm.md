+++
title = "Troubleshooting Terraform Kubernetes and Helm deployment"
description = "Troubleshooting Terraform Kubernetes and Helm deployment"
tags = [
    "terraform",
    "helm",
    "kubernetes",
    "devops",
    "automation"
]
date = "2019-09-23"
categories = [
    "Development",
    "Automation",
    "Kubernetes",
    "Terraform",
    "Helm"
]
+++

I decided to rebuild my home lab Plex server using Kubernetes and a great Helm chart (https://github.com/munnerz/kube-plex) which
dispatches transcode jobs as pods on the Kubernetes cluster. I wanted to do this in a fully automated fashion so that I could
destroy and rebuild the whole infrastructure with one command thus saving me precious pennies and preventing any snowflake environments
forming.

I used Terraform to build the Azure Kubernetes Service (AKS) and all was going well until I tried integrating the Terraform
Helm resources into my Terraform code. The AKS cluster was building absolutely fine but as soon as it got to the Helm resource,
it immediately bombed with the following error:

```powershell
Error: error installing: Post https://k8stest-aa211a06.hcp.ukwest.azmk8s.io:443/apis/extensions/v1beta1/namespaces/kube-system/deployments: 
dial tcp 92.242.132.15:443: connectex: A connection attempt failed because the connected party did not properly respond after a period of time,
or established connection failed because connected host has failed to respond.
```

Reading the error at face value, it looked like the Helm resource couldn't connect to my Kubernetes cluster on the hostname `k8stest-aa211a06.hcp.ukwest.azmk8s.io`.
This was strange because using `kubectl cluster-info` showed the cluster up and running:

```powershell
Kubernetes master is running at https://k8stest-bbf7a87b.hcp.ukwest.azmk8s.io:443
CoreDNS is running at https://k8stest-bbf7a87b.hcp.ukwest.azmk8s.io:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://k8stest-bbf7a87b.hcp.ukwest.azmk8s.io:443/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy
Metrics-server is running at https://k8stest-bbf7a87b.hcp.ukwest.azmk8s.io:443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

If you're eagle eyed, you might have already spotted the issue here but at this point, I was still scratching my head.
As Terraform is idempotent, I ran a `terraform apply --auto-approve` to try everything again. As the Cluster was already up and working,
Terraform would realise this and only try and deploy the Helm resource again:

`Apply complete! Resources: 1 added, 0 changed, 0 destroyed.`

Err, right. So all I've done is reapply the same Terraform configuration and now my Helm chart has successfully deployed to my AKS cluster?
At this point I thought that perhaps my local kubectl environment hadn't been set up to point to the new AKS cluster I was created. I added
a local_exec resource to initialise the environment with my AKS credentials just before I kick off the Helm resource:

```bash
resource "null_resource" "initialise_kubectl" {
  provisioner "local-exec" {
    command = "az aks get-credentials --resource-group ${var.resource_group_name} --name ${var.cluster_name} --overwrite-existing"
  }

  depends_on = [azurerm_kubernetes_cluster.k8s, azurerm_public_ip.plex_publicip]
}
```

I destroyed my current Terraform environment and reran it and unfortunately got the same error:

```powershell
Error: error installing: Post https://k8stest-bbf7a87b.hcp.ukwest.azmk8s.io:443/apis/extensions/v1beta1/namespaces/kube-system/deployments: 
dial tcp 92.242.132.15:443: connectex: A connection attempt failed because the connected party did not properly respond after a period of time,
or established connection failed because connected host has failed to respond.
```

Right, let's just check whether this hostname is resolvable:

```powershell
Pinging k8stest-bbf7a87b.hcp.ukwest.azmk8s.io [92.242.132.15] with 32 bytes of data:
Request timed out.
Request timed out.
Request timed out.
```

Nope. However, we know the Cluster has built successfully so this FQDN should definitely have an IP associated with it.
Let's jump over to PowerShell and check what the Cluster looks like:

```powershell
Get-AzAks

ProvisioningState       : Succeeded
DnsPrefix               : k8stest
Fqdn                    : k8stest-134c5320.hcp.ukwest.azmk8s.io
KubernetesVersion       : 1.13.10
AgentPoolProfiles       : {agentpool}
LinuxProfile            : Microsoft.Azure.Commands.Aks.Models.PSContainerServiceLinuxProfile
ServicePrincipalProfile : Microsoft.Azure.Commands.Aks.Models.PSContainerServiceServicePrincipalProfile
Id                      : /subscriptions/ed31d49b-a568-490a-8ee2-0cbaec65bc9b/resourcegroups/azure-k8stest/providers/Microsoft.ContainerService/managedClusters/k8stest
Name                    : k8stest
Type                    : Microsoft.ContainerService/ManagedClusters
Location                : ukwest
Tags                    : {[Environment, Development]}
```

Hold up a minute, that FQDN isn't the same one that Helm was trying to connect on. As everything
in our code is dynamically generated, where is it getting this hostname from? I was scratching my head for a while
and then saw that the hostname it's trying to connect to is actually the hostname of the previous build. So the hostname
must be cached somewhere and it's picking up that one instead of the correct one. Looking at the config file in the `.kube` folder in
my home directory, it was showing the correct hostname but within the `.kube` folder there's a `cache` folder that contains sub-folders
of all of your previous builds so I started wondering whether it was picking up a cached hostname.

Just in case this was a weird caching issue, I decided to delete my .kube folder because the AKS Terraform resource would recreate it as part of the build anyway but when
I tried to run my `terraform apply` again, I got the following error:

```powershell
Error: CreateFile C:\Users\matth\.kube\config: The system cannot find the path specified.
```

With the above error, Terraform didn't even try to apply the configuration. The next step was to comment out the Helm section of
the Terraform and low and behold; no compilation error.

At this point it became pretty obvious what was going on. The Helm resource grabs your Kubernetes Cluster information from the `config` file in the `.kube` folder
at the very start of your Terraform build. I was trying to dynamically create my Kubernetes config during the build
with that `local-exec` command I mentioned earlier. Helm was basically picking up whatever was already in my Kubernetes config file
and trying to connect to that. It just so happened that I'd ran previous AKS builds and with each dynamic hostname being so similar,
took a bit of digging around to get to the root cause.

## Solution

The quickest solution to this is to separate out the AKS configuration from the Helm configuration and ensure the Kubernetes config file
is up to date with my latest AKS details *before* I run the Helm resource.

However, a much better way to sort this is to configure the Terraform Helm Provider to accept the correct Kubernetes settings. The
`azurerm_kubernetes_cluster` resource has a bunch of useful outputs that we can use to dynamically initialise Helm after our fresh Kubernetes
cluster has been built:

```bash
provider "helm" {
  kubernetes {
    host                   = azurerm_kubernetes_cluster.aks_cluster.host
    client_certificate     = base64decode(azurerm_kubernetes_cluster.aks_cluster.client_certificate)
    client_key             = base64decode(azurerm_kubernetes_cluster.aks_cluster.client_key)
    cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.aks_cluster.cluster_ca_certificate)
  }
}
```

In the above code, we're telling Helm to use the values from our brand new Kubernetes cluster resource as opposed to just loading
the default Kubernetes config file which it was doing before we had the Helm provider configured.

One thing to note with providers is that as of Terraform v0.12, you can't use a `depends_on` to force it to wait for a
specific resource. Luckily for us, Terraform works best with implicit dependencies - as we've specified values that have come
directly from our Kubernetes resource, the Helm provider will wait until it's complete until it initialises.

### Hacky tip of the day

If you ever need an explicit dependency for your Provider, I stumbled across this cool little work-around solution somebody
suggested on GitHub: https://github.com/hashicorp/terraform/issues/2430#issuecomment-524547219 
