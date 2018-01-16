+++
title = "Error when running 'vagrant up'"
description = ""
tags = [
    "powershell",
    "automation",
    "vagrant",
    "virtualbox"
]
date = "2018-01-16"
categories = [
    "Development",
    "Automation",
    "PowerShell"
]
menu = "main"
+++

I've been using vagrant to build up labs for my work recently and it's an awesome product. One small downside is the sheer amount of space that the new VMs were taking up on my laptop SSD (I'd been testing linked clones and hadn't cleaned up after myself). As my whole lab build was in code, I felt rather pleased with myself that I could just bin the whole environment including any VMs and then just spin it up again afterwards. So, off I went to the VirtualBox VM directory ("$env:userprofile\VirtualBox VMs"). When I looked at the total size of the directory, it was over 100GB in space and because being heavy handed works so well in I.T (Hmm...), I deleted all of the VMs in the folder.

Right, time to 'vagarnt up' my lab and sit back and watch the infrastructure as code goodness: 

![](/vagrant-fail.PNG)

AH! Perhaps my heavy handed approach probably wasn't the best idea. It seems like there might be some sort of lock on the files Vagrant is trying to use. Wait, there shouldn't be any files needed because they've all been deleted, right? Looking in VirtualBox shows that it still thinks the VMs exist but are inaccessible (I forgot to grab a screenshot, sorry!). To fix this, I deleted the inaccessible VMs from within VirtualBox and ran a 'vagrant up' again and it imported a fresh box with no errors. 

As I'm using linked clones for my environment, Vagrant must have been looking for the initial clone that my lab was supposed to be using (The one that I'd heavy handedly deleted). As VirtualBox still thought it had the clone in existence, Vagrant was spitting out errors when trying to build the lab. 

