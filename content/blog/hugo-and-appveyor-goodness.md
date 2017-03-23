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

I won't go into too much detail about Hugo because there's some great stuff online already but essentially, Hugo renders all of your static content into HTML, CSS and JavaScript. This makes pages on the site really quick to load and easy to manage as you don't rely on a traditional CMS to manage content. 
<br><br>
Install Hugo by running:
```powershell
choco install hugo -y
```

Next up, creating a new skeleton site structure is as simple as running:
```powershell
hugo new site mynewblog
```

If you have a look in your newly created site folder, it'll look something like this: 

![](/hugo-file-structure.PNG)

Now that we have the basic layout, we need to add a theme. I ended up browsing the [theme section](http://themes.gohugo.io/) on Hugo's website to find one I liked. Once you've found a theme that you like, clone it into the themes folder of your newly created site.

On my local machine I use posh-git which integrates Git nicely into PowerShell (Including tab completion). To install posh-git, simply run the following from PowerShell:
```powershell
choco install poshgit -y
```
*Note: You may need to reload your PowerShell session to use the Git commands from PowerShell.*
<br><br>
With Git installed, clone the theme you like:

```powershell
cd themes
git clone https://github.com/example/example-theme
```

Usually, a theme with have an 'exampleSite' directory which contains an example structure for the theme including an example config.toml configuration file: 

![](/theme-example-site.PNG)

I copied these files over into the root of my new Hugo site and started tailoring to fit my needs. I won't go into any further detail on that part - just play around with it and follow your nose.
<br><br>
So, you've got a theme you're happy with and you've tweaked the configuration to your liking? Hugo has a very convenient webserver that listens on port 1313 so that you can view how your glorious website will look when built. To make use of this, run the command:
```powershell
hugo server  
```
The great thing about this is, everytime you update any of your static files, the changes will instantly appear at http://localhost:1313
<br><br>
Right, now we've got our new Hugo site configured the way we want it, let's set up our Github pages.

## Step 2. Setting up Github Pages to work with our new Hugo site

If you haven't already got a Github account, head over to [GitHub](https://github.com) and sign up for one. Once you've got your account set up, create a new repository and call it yourusername.github.io e.g. mine is matthorgan.github.io. 

For this build, we're going to be using two branches - 'source' which will house all of the static content and 'master' which will house the Hugo built content. Doing it this way allows us to have both the static and generated content completely version controlled.
<br><br>
From PowerShell, browse to the root directory of your Hugo site and run the following commands to set up your local repository and push it to your Github repo:
```powershell
git init
git add --all
git commit -m "Initial Hugo static file commit"
git remote add origin https://github.com/yourusername/yourusername.github.io.git
git push -u origin source
``` 
*Note: Replace 'yourusername' with your actual Github username and repo name*
<br><br>
We've now got our static content pushed to a branch in our Github Pages repo called 'source'. At this point, you can move on to the continuous integration with AppVeyor however just for completeness, I'll continue to explain how you get your page online without a CI tool. 

Type the following command from a PowerShell prompt in the root directory of your Hugo site to build the site:
```powershell
hugo
```

If you have a look in the root directory, you'll now see a 'public' folder which has all of your built content e.g. HTML, JavaScript, CSS etc.

Change directory into the public folder and type the following:
```powershell
git checkout -b master
git add --all
git commit -m "Initial Hugo built site commit"
git push -u origin master 
```

You should now have your website online and browsable at https://yourusername.github.io - take a moment to bask in its unfinished glory before we move on to the cool stuff! 

## Step 3. Using AppVeyor to automate the Hugo build

Having our newly generated website up and running already is cool but each time you add/modify any of the static content, the site will need to be rebuilt and pushed up to GitHub. That's at least two buttons more than I want to have to press so we're going to use AppVeyor to automate it for us.


## Step 4. Setting up a custom domain with GitHub Pages (Optional)