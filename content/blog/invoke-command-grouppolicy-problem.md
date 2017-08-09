+++
title = "Invoke-Command Group Policy Gotcha (Whoops!)"
description = ""
tags = [
    "powershell",
    "automation",
    "invoke-command",
    "gpupdate /force"
]
date = "2017-07-09"
categories = [
    "Development",
    "Automation",
    "PowerShell"
]
menu = "main"
+++

A task that I've been working on recently is the automation of VM builds and post configuration using PowerShell and the PowerCLI module (Unfortunately we don't have a configuration management tool to make our lives easier at the moment).

Part of the post-build configuration is to run a bunch of commands on the newly built VMs. I could have used PowerCLI's Invoke-VMScript command to achieve this however as all of the VMs were joining the domain and tools on some of the VM templates were out of date, I used PowerShell's Invoke-Command. This was going swimmingly until I ran into an issue...  

The final requirement of the initial post configuration was to force refresh Group Policy on the machine. Easy right?  

```powershell
Invoke-Command -ComputerName $VmName -ScriptBlock {gpupdate /force}
```
Upon running this command, I was faced with absolutely nothing being returned - the prompt was just hanging. Eh?! So, what happens if I just run a gpupdate without the force switch?  

![](/gpupdate-works.PNG)

Right, so that works perfectly. Hmm, maybe the sheer number of policies that need to apply is slowing things down? What happens if we try a VM that already has the right policies? 

![](/gpupdate-works.PNG)

OK, so this seems to imply that the issue is purely with the inital application of the policies. The next step is to RDP onto the VM and try the command locally to see what gets returned:

![](/gpupdate-requires-reboot.PNG)

Bingo! The Group Policy update was working fine but required a reboot. Of course Invoke-Command isn't going to return anything if the remote command it's running is sat waiting for user input.  

## Resolution
Armed with the above information, the solution was rather simple - gpupdate has a /boot flag which reboots the VM if required. So all that we needed to get Invoke-Command working correctly was:
```powershell
Invoke-Command -ComputerName $VmName -ScriptBlock {gpupdate /force /boot}
```
The take-away here is to always consider that a particular command might not behave the same way you've seen it behave on other machines in the past. As a side note, if there wasn't a /boot switch then you could probably spoof the user input required by doing something like:  
```powershell
Invoke-Command -ComputerName $VmName -ScriptBlock {echo "Y" | gpupdate /force}
```