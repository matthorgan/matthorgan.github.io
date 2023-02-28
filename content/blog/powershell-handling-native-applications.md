+++
title = "Handling Native Applications in PowerShell 7.3+ with $PSNativeCommandErrorActionPreference"
description = ""
tags = [
    "powershell",
    "automation",
    "devops"
]
date = "2023-02-28"
categories = [
    "Development",
    "Automation",
    "DevOps"
]
+++

I've previously written a [blog post](azcli-error-handling-in-powershell.md) about how to handle Azure CLI errors in PowerShell. The general pattern involves redirecting any error streams to the correct place, and checking for a known exit code. There's an exciting new feature in PowerShell from version 7.3 onwards that makes capturing errors for native applications like the Azure CLI much simpler. Enter: `$PSNativeCommandErrorActionPreference`

[PSNativeCommandErrorActionPreference](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_preference_variables?view=powershell-7.3#psnativecommanduseerroractionpreference) is a preference variable that when set to `true`, allows for native commands to be handled in a more 'PowerShell' way.

To enable it, you just need to set `$PSNativeCommandErrorActionPreference = $true` somewhere in your script, and then you can use a conventional try/catch for your error handling. Check out this before/after example:

### Before

1. Assign variable to output
1. Redirect the error stream and assign to variable `$err` to ensure any error(s) get captured
1. If the `$LASTEXITCODE` isn't a success, throw the contents of the `$err` variable

```powershell
$err = $($podList = kubectl get pods) 2>&1
if ($LASTEXITCODE -ne 0) {
    throw $err
}
```

### After

1. Set the `$PSNativeCommandUseErrorActionPreference` to `true`
1. Set `$ErrorActionPreference` to `Stop`
1. Handle the error in a traditional try/catch and it'll throw if there's a non-zero exit code (Note you may still want to redirect the error stream depending on the command)

```powershell
$PSNativeCommandUseErrorActionPreference = $true
$ErrorActionPreference = 'Stop'

try {
    $podList = kubectl get pods
} catch {
    throw
}
```
