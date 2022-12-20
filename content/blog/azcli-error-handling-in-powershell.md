+++
title = "Azure CLI Error Handling in PowerShell"
description = ""
tags = [
    "powershell",
    "automation",
    "azcli",
    "ci/cd",
    "devops"
]
date = "2021-09-16"
categories = [
    "Development",
    "Automation",
    "DevOps"
]
+++

The Azure CLI (azcli) is an extremely useful tool in interacting with the Azure platform. I've been favouring it over the Azure PowerShell modules recently due to many of the commands being idempotent (E.g. not having to check if something already exists before trying to create can be a nice time saver). One of the downsides to using the azcli in PowerShell scripts however, is that you can't handle errors like you would with a typical PowerShell cmdlet.

Before we get into the code, it's worth saying that to filter data with the Azure CLI, you've got two options. Option one is to use the JMESPath query parameter e.g. in the following example `az vm show --resource-group QueryDemo --name TestVM --query "osProfile.linuxConfiguration.ssh.publicKeys"`. The default output from the Azure CLI is JSON and so this query is targeting a nested property `publicKeys` which in JSON would look something like `"osProfile:" { "linuxConfiguration": {"ssh": { "publicKeys": [{ "someData": "someDataHere" }]} } }`. The option I prefer however, is to use PowerShell's `ConvertFrom-Json` cmdlet which brings us into our warm and cosy PowerShell world. So the same filter would be: `(az vm show --resource-group QueryDemo --name TestVM | ConvertFrom-Json).osProfile.linuxConfiguration.ssh.publicKeys`. I'll be adding the `ConvertFrom-Json` cmdlet to the examples as this is more representitive of how I'd actually use the azcli.

Consider the below examples with an App Registration that doesn't exist:

```powershell
try {
    $appReg = Get-AzureADApplication -ObjectId 'NotARealObjectId' -ErrorAction 'Stop'
}
catch {
    Write-Error "Uh oh, we've got an error here..."
    throw
}
```

```powershell
try {
    $appReg = az ad app show --id 'NotARealObjectId' | ConvertFrom-Json
}
catch {
    Write-Error "Uh oh, we've got an error here..."
    throw
}
```

In the first example, we get a lovely handled error and a custom message but in the azcli example, nothing gets thrown because the azcli is an external tool and PowerShell doesn't natively know what to do with it.

We can improve things by making use of the `$LASTEXITCODE` which will give you the exit code of the last ran command.

```powershell
$appReg = az ad sp show --id 'NotARealObjectId' | ConvertFrom-Json
if ($LASTEXITCODE -ne 0) {
    Write-Error "Uh oh, we've got an error here..." -ErrorAction 'Stop'
}
```

The above example will display our custom error message and halt the script if we get an exit code that isn't 0. You'll also find that the azcli spits out its own error but at the moment this isn't information that we can capture within the constraints of our own error handling - it'll just show the error as soon as the command isn't successful.

So how do we capture the azcli error within our own error handling? We can utilise a bit of stream redirection and a subexpression to achieve a better result:

```powershell
$errOutput = $($appReg = & {az ad sp show --id 'NotARealObjectId' | ConvertFrom-Json}) 2>&1
if ($errOutput) {
    Write-Error "Uh oh, we've got an error here..." -ErrorAction 'Continue'
    throw $errOutput
}
```

In this final example, we're executing the azcli command using the ampersand operator and capturing the variable output in the `$appReg` variable like before. However, this time we're using a subexpression and redirecting the error stream of that subexpression to the success stream and capturing it in a variable called `$errOutput`. Now that we've got the error in a variable, we can display it and handle it however we like.

One thing to note is make sure you're aware of what output the azcli command you're using gives you. Some azcli commands send useful non-error commands to the error stream and therefore redirecting it like we've done above will trigger an error. If you wanted to, you could technically use a hybrid of the above two methods to ensure this doesn't trip you up:

```powershell
$errOutput = $($appReg = & {az ad sp show --id 'NotARealObjectId' | ConvertFrom-Json}) 2>&1
if ($LASTEXITCODE -ne 0) {
    Write-Error "Uh oh, we've got an error here..." -ErrorAction 'Continue'
    throw $errOutput
}
```
