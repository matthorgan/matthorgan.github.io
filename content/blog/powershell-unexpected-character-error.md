+++
title = "PowerShell Unexpected Character Error"
description = ""
tags = [
    "powershell",
    "automation",
    "azure",
    "ci/cd",
    "devops"
]
date = "2023-01-05"
categories = [
    "Development",
    "Automation",
    "DevOps"
]
+++

Over the years, I've had a few different `Unexpected character encountered` errors in PowerShell and for the longest time, they were always a real pain to troubleshoot and looked like a bug. Once you know the issue though, it's a fairly simple one to solve. The latest one I've come across is running the following command from our CI/CD Linux build container to a Windows 2012 machine hosted in Azure:

`Invoke-AzVMRunCommand -ResourceGroupName $VMResourceGroupName -Name $VMName -CommandId 'RunPowerShellScript' -ScriptPath "tests/gather-vm-info.ps1"`

The above command is using the Azure VM run command to run the contents of `tests/gather-vm-info.ps1` on the remote Windows 2012 VM. It should be noted, I was running the exact same command on Windows 2016, and Windows 2019 VMs without any issues. The error I was seeing was the following:

```json
JsonReaderException: Unexpected character encountered while parsing value: W. Path '', line 0, position 0.
ArgumentException: Conversion from JSON failed with error: Unexpected character encountered while parsing value: W. Path '', line 0, position 0.
at <ScriptBlock>, /builds/golden-images/tests/gather-vm-info.ps1:49
```

Initially looking at this error, it felt 'buggy' considering there was no 'W' in my output when I ran the `tests/gather-vm-info.ps1` code individually on the VM itself. I assumed it must be an issue with the `Invoke-AzVMRunCommand` but actually, when you look closer at the output, it's easy to see what the issue is.

```
Value[0]        : 
  Code          : ComponentStatus/StdOut/succeeded
  Level         : Info
  DisplayStatus : Provisioning succeeded
  Message       : WARNING: Ping to <redacted URL> failed -- Status: TimedOut
{
    "Output1":  true,
    "Output2":  true,
    "Output3":  true,
    "Output4":  true,
    "Output5":  false,
    "Output6":  true,
    "Output7":  true,
    "Output8":  false,
    "Output9":  false
}
```

As you can see, on investigating the `Value.Message` property of the object returned from `Invoke-AzVMRunCommand`, we've got a warning message at the very start of the output before our custom JSON. Suddenly, the `W` character in the initial error makes sense. In our scenario, `Value.Message` returns all output including any warning messages and therefore whilst our JSON is being returned correctly, we need to ensure that any warning messages are properly handled or suppressed to ensure that our output only returns JSON ready to be converted.

So, the next time you see a weird unexpected parsing error, have a look at what the starting character is. If it's a `W`, there's a good chance you could be trying to parse something that has captured one of the PowerShell output streams along with your expected output.
