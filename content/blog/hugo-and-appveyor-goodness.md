+++
title = "Hugo & AppVeyor (Continuous Integration Blog Goodness)"
description = ""
tags = [
    "go",
    "golang",
    "hugo",
    "development",
    "powershell",
    "winops",
    "continuous integration",
    "appveyor"
]
date = "2017-03-16"
categories = [
    "Development",
    "golang",
    "WinOps"
]
menu = "main"
+++

Recently, I've been thinking about how I can use DevOps tools and processes to improve the way I do things and this has led me to completely revamp my blog. My existing Wordpress blog has now been replaced with a shiny new blog using [Hugo](https://gohugo.io/) - a static website engine, hosted on [Github Pages](https://pages.github.com/) and automated using the continuous integration tool [AppVeyor](https://www.appveyor.com/). In this article I'm going to go through a step-by-step guide on how I set everything up. 

## Step 1. Install Hugo & Build New Site
First off, it's worth mentioning that this step-by-step guide assumes you're running a Windows environment and using the Windows package manager Chocolatey (If you don't know about Chocolatey then you NEED to get this in your life right now. Check it out [here](https://chocolatey.org/)). 

Install Hugo by running:
```
choco install hugo -y
```

Next up, creating a new skeleton site structure is as simple as running:
```
hugo new site mynewblog
```

If you have a look in your newly created site folder, it'll look something like this: 

![](/hugo_file_structure.PNG)

-- To be continued --