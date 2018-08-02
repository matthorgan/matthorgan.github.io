+++
title = "Register-SCDWSource Cmdlet hidden parameter"
description = ""
tags = [
    "powershell",
    "automation",
    "service manager",
    "system center"
]
date = "2018-08-01"
categories = [
    "Development",
    "Automation",
    "PowerShell",
    "System Center Service Manager"
]
menu = "main"
+++

Myself and a colleague were recently given the task of end-to-end automation of Sytem Center Service Manager using
PowerShell DSC. This needed to include prerequisites, installation of the product, and any post configuration.

If any of you are familiar with Service Manager, you'll know that one of the steps is to register Service Manager to the
data warehouse. Luckily, there's a PowerShell cmdlet for that which should make the task a breeze: [Register-SCDWSource](https://docs.microsoft.com/en-us/powershell/module/microsoft.enterprisemanagement.warehouse.cmdlets/register-scdwsource?view=systemcenter-ps-2016)

Looking at the documentation, I've modified Microsoft's example to use my credentials and VM names to register the datawarehouse with Service Manager:

```powershell
$creds = New-Object -TypeName 'PSCredential' -ArgumentList ('lab\Administrator', (ConvertTo-SecureString -String 'MyLabPassword' -AsPlainText -Force))
Register-SCDWSource -ComputerName 'scsmdw1' -SourceComputerName 'scsmms1' -DataSourceTypeName 'ServiceManager' -Credential $creds
```

When you run this command, a credential request pop-up box appears which would imply that the $creds variable hasn't worked hence the prompt. Looking back
at the example online, there's nothing obvious missed and the $creds variable contains a credential object as you'd expect.

![](/cred_popup1.PNG)

Upon cancelling the prompt pop-up box, things started becoming clear:

![](/cred_popup2.PNG)

I should have paid more attention to the initial message behind the pop-up box - the -Credential parameter and $creds variable
were working exactly as expected BUT the `Register-SCDWSource` cmdlet actually required the `SourceCredential` parameter too.
I tried to use the auto-complete functionality for the `SourceCredential` parameter but nothing was coming up (and nothing was mentioned about this parameter in the documentation) which filled me with 0% confidence. However, I ended up trying it and to my shock, the cmdlet worked a treat:

```powershell
$creds = New-Object -TypeName 'PSCredential' -ArgumentList ('lab\Administrator', (ConvertTo-SecureString -String 'MyLabPassword' -AsPlainText -Force))
Register-SCDWSource -ComputerName 'scsmdw1' -SourceComputerName 'scsmms1' -DataSourceTypeName 'ServiceManager' -Credential $creds -SourceCredential $creds
```

A great lesson here is that we're all human and even Microsoft are going to make mistakes with their documentation and cmdlets. Hopefully the feedback will ensure that they update the documentation and if not then, hopefully this post helps!
