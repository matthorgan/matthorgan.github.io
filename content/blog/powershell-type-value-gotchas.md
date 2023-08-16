+++
title = "Handling null values in PowerShell and the sneaky gotcha that might catch you out"
description = "Handling null values in PowerShell and the sneaky gotcha that might catch you out"
tags = [
    "powershell",
    "automation",
    "ci/cd",
    "devops"
]
date = "2022-09-15"
categories = [
    "Development",
    "Automation",
    "DevOps"
]
+++


In this blog post, I'll be covering a fun little null value related 'gotcha' that has caught me out over the years, and the explanation
for why it happens. Let's jump right in with how we handle null values and the issues I've come across in the past.

## Handling Null Values

In PowerShell, there are several ways to check for a null value in a variable. One of the standard ways looks like this:

```powershell
$MyVariable = $null

if ($null -eq $MyVariable) {
    "The variable is null"
}

The variable is null
```

Another option is to use the static .NET String class and call the `IsNullOrEmpty` method like this:

```powershell
$MyVariable = $null

if ([String]::IsNullOrEmpty($MyVariable)) {
    "The variable is null"
}

The variable is null
```

Or, if you want to handle whitespace too, you could use the `IsNullOrWhiteSpace` method:

```powershell
# Whitespace variable that we want to check for
$MyVariable = ' '

if ([String]::IsNullOrWhiteSpace($MyVariable)) {
    "The variable is null or is whitespace"
}

The variable is null or is whitespace
```

## The Null String 'Gotcha'

A lot of the time, I found myself opting for the `IsNullOrWhiteSpace` option because I'd previously been caught out by
`$null -eq $MyVariable` not giving me the result I was expecting. I never fully understood why until recently when I was
caught out once again.

A significant portion of my PowerShell scripts and functions are used in CI/CD pipelines, where environment variables
inherited from the CI/CD tool, such as a hostname or username, can be present. Instead of explicitly passing every
variable into my scripts/functions, I use a pattern like this:

```powershell
function New-ExampleFunction {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        [String]$MyVariable = $env:VAR_FROM_CI_ENVIRONMENT
    )
    
    Write-Host $MyVariable
}
```

As seen in the above function, the environment variable `VAR_FROM_CI_ENVIRONMENT` is available within the pipeline
of our CI/CD tool. Therefore, we don't need to provide a value for `$MyVariable` explicitly in this case.
This approach offers the flexibility of passing a value to the parameter if desired, while defaulting to the known
environment variable otherwise.

The issue however, is that outside the scope of our CI/CD environment, what happens if somebody doesn't pass in a value
for `$MyVariable` and they don't have the `$env:VAR_FROM_CI_ENVIRONMENT` environment variable configured? We need to add
an additional bit of validation in our function to ensure that `$MyVariable` contains a value if one isn't passed in
(The `[ValidateNotNullOrEmpty()]` is only validating values being passed in to $MyVariable, and not something we set as a default value).

Adding some validation to the above function to check for null can create a nasty little gotcha and this is the main topic
for this blog post. I added a traditional `$null -eq $MyVariable` check to ensure if `$MyVariable` was null and expected
that it would throw a message and stop the function (Assuming nothing
had been passed in OR the `$env:VAR_FROM_CI_ENVIRONMENT` environment variable hadn't been set):

```powershell
# This does not produce the expected outcome
function New-ExampleFunction {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        [String]$MyVariable = $env:VAR_FROM_CI_ENVIRONMENT
    )

    if ($null -eq $MyVariable) {
        throw 'The value for parameter "MyVariable" is blank. Please pass in a value or check the env var $env:VAR_FROM_CI_ENVIRONMENT is set'
    }
    
    Write-Host $MyVariable
}
```

When you run the above function without passing in a value for `$MyVariable`, and ensuring the `VAR_FROM_CI_ENVIRONMENT`
env var is not set, nothing happens and we don't see the error message we're expecting.

Well that's not what I was expecting?! Checking that the env var is definitely not set by running `$null -eq $env:VAR_FROM_CI_ENVIRONMENT`
comes back as `True` so it definitely IS null. Why isn't PowerShell recognising it as null?

If we change how we're checking for null to use the static string class method, look what happens:

```powershell
# This DOES produce the expected outcome
function New-ExampleFunction {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        [String]$MyVariable = $env:VAR_FROM_CI_ENVIRONMENT
    )

    if ([String]::IsNullOrEmpty($MyVariable)) {
        throw 'The value for parameter "MyVariable" is blank. Please pass in a value or check the env var $env:VAR_FROM_CI_ENVIRONMENT is set'
    }
    
    Write-Host $MyVariable
}
```

```text
Exception: 
Line |
  14 |          throw 'The value for parameter "MyVariable" is blank. Please  …
     |          ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     | The value for parameter "MyVariable" is blank. Please pass in a value or check the env var $env:VAR_FROM_CI_ENVIRONMENT is set
```

This is now working as expected. But why? In the past I used to just change the `$null -eq $MyVariable` syntax to
`[String]::IsNullOrWhiteSpace()`, wrongly assuming that the variable must have been whitespace but as you can see from
the above working code, we're using `[String]::IsNullOrEmpty()` so that theory has been proved to be incorrect.

Look what happens when we remove the `[String]` casting from the function and we go back to using our `$null -eq $MyVariable`
syntax:

```powershell
function New-ExampleFunction {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        # Note the [String] has now been removed
        $MyVariable = $env:VAR_FROM_CI_ENVIRONMENT
    )

    if ($null -eq $MyVariable) {
        throw 'The value for parameter "MyVariable" is blank. Please pass in a value or check the env var $env:VAR_FROM_CI_ENVIRONMENT is set'
    }

    Write-Host $MyVariable
}
```

We now see the error message that we were hoping to see originally (and that we saw with the `[String]::IsNullOrEmpty()` method):

```text
Exception: 
Line |
  14 |          throw 'The value for parameter "MyVariable" is blank. Please  …
     |          ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     | The value for parameter "MyVariable" is blank. Please pass in a value or check the env var $env:VAR_FROM_CI_ENVIRONMENT is set
```

### Conclusion

Alrighty then. So it seems that casting the parameter `MyVariable` to a string is causing our original null comparison to fail.
The reason for this is that when we add the `[String]` to our parameter `$MyVariable` and we don't pass in a value, it
takes the `$env:VAR_FROM_CI_ENVIRONMENT` and converts it into an empty string. When you strongly type a value type in PowerShell,
it will convert `$null` into whatever the default value is for the type:

```powershell
[int]$number = $null
$number
0

[bool]$boolean = $null
$boolean
False

[string]$string = $null
$string -eq ''
True
```

This explains the initial failure of our `$null -eq $MyVariable` example. PowerShell is converting the empty `$MyVariable` into
the default value for a string and an empty string is not truly null.
Using the string class method `[String]::IsNullOrEmpty()` ensures proper handling of an empty string.

When `$MyVariable` wasn't strongly typed to any specific type, the `$null -eq $MyVariable` comparison worked because,
at that point, PowerShell didn't know the type and therefore didn't convert it to a default value of any specific type

And there we have it. I can finally sleep a little easier now that I have a clear picture of the potential
type-related issues when evaluating `$null` and why they occur.
