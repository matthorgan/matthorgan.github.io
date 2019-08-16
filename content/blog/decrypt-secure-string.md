+++
title = "Decrypting PowerShell Secure Strings"
description = "Quick tip for Secure String Decryption"
tags = [
    "powershell",
    "automation"
]
date = "2019-08-15"
categories = [
    "Development",
    "Automation",
    "PowerShell",
]
menu = "main"
+++

Ever found yourself needing to decrypt a secure string knowing that there are a couple of static methods you need to use
but can never remember what they are? After a quick Google, you'll probably stumble upon something similar to this:

```powershell
$BSTR = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecurePassword)
$UnsecurePassword = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)
```

Whilst this works fine, it's way easier to remember (Tab complete is your friend) that a `PSCredential` object has the 
ability to show the password by using `GetNetworkCredential().Password`. So, you can construct a PSCredential object
with arbitrary username (I used a whitespace) and access the secure string like so:

```
$UnsecurePassword = [PSCredential]::New(' ', $SecurePassword).GetNetworkCredential().Password
```