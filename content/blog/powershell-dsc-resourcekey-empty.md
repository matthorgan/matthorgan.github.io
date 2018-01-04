+++
title = "PowerShell DSC ResourceKey Empty"
description = ""
tags = [
    "powershell",
    "automation",
    "dsc",
    "sql"
]
date = "2018-01-04"
categories = [
    "Development",
    "Automation",
    "PowerShell",
	"DSC",
	"SQL"
]
menu = "main"
+++

I'm absolutely loving PowerShell DSC at the moment and we're using it heavily for the automation and configuration of various products. One of the most recent tasks was to create a fully parameterised SQL build function for reusability across the company. All was going swimmingly until I ran into the following error when attempting to run my freshly parameterised DSC function: 

![](/initial_dsc_error.PNG)

After a little bit of digging, I stumbled across a great blog post by Jacob Benson which had some nice troubleshooting steps ([Link here](http://jacobbenson.com/?p=735#sthash.zRNaYRyN.dpbs)). This blog pointed me in the right direction - the key property of the DSC resource always has to be present. Well, weirdly, all of my parameters were being correctly passed through because I'd verified that in a verbose message:

![](/verified_parameters.PNG)

I had a quick look in the MOF file for the DSC resource I was using (SqlSetup in SqlServerDsc)and noted that the key was 'InstanceName'. I then changed the InstanceName parameter to a hardcoded value and re-ran the configuration... BINGO! It worked absolutely fine which meant there must have been something wrong with my InstanceName parameter. 

A DSC configuration will automatically have three default parameters when it is created - InstanceName, OutputPath, and ConfigurationData. After rolling back to a fresh snapshot in my lab, I decided to change the name of the parameter I was using for InstanceName because I'd actually just called it $InstanceName for simplicity. I renamed it to $SqlInstanceName and re-ran the configuration and SQL installed like a charm. 

I've never had to use the default parameter 'InstanceName' for any of my configurations so it took me a while to realise what was going wrong. However, when you think that any variables with the same names as default parameter names are going to be overwritten with the default parameter values, it makes perfect sense. The original error wasn't misleading at all - 'ResourceKey' WAS an empty string and next time I'll be more careful with my function parameter names to ensure they don't clash with any DSC configuration default parameters.

