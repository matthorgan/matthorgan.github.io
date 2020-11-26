+++
title = "Moving from AppVeyor to GitHub Actions"
description = "Moving from AppVeyor to GitHub Actions"
tags = [
    "appveyor",
    "github",
    "actions",
    "devops",
    "ci/cd"
]
date = "2020-11-26"
categories = [
    "Development",
    "DevOps",
    "CI/CD"
]
+++

It's been over three and a half years since I first created my blog with Hugo and AppVeyor. Since then, I've been fortunate (or unfortunate depending on the tool!) to use lots of other CI/CD tools such as GitLab CI, Jenkins, Bamboo, Azure DevOps. GitHub Actions is one tool that I hadn't used yet and it's been on my radar for a while so why not take the opportunity to update the blog and the CI pipeline?

One of the main wins for me with GitHub Actions is the integration with GitHub. Having your code and your CI pipeline in one familiar GUI makes for a really nice experience. There's also a great VSCode plugin I found called `cschleiden.vscode-github-actions` which provides a really nice view of all your pipeline builds (or workflows as they're called in GitHub) like this:

![](/img/ghactions-vscode-extension.PNG)

Another great benefit of GitHub Actions is the marketplace that contains thousands of actions that can help automate your workflow.

The first thing I was going to do before I got started with GitHub Actions was to move my old inline code from the AppVeyor build file and into its own script. Over the years, I've realised that pipelines can get wildly out of hand if you don't abstract as much code away from the pipeline syntax as possible. This makes your pipelines way easier to read and understand and allows for them to be easily portable into other CI tools if required.

After some initial testing with the Windows 2019 runner, I decided to swap over to Linux as the builds went from around 30s in AppVeyor to a whole 3m30 in GitHub Actions.

Before I started to refactor my code into a little bash script, I thought I'd have a quick look at what community Actions were available to potentially simplify my workflow and lo and behold, it couldn't have been simpler.

There's a Hugo Action available from [here](https://github.com/peaceiris/actions-hugo) which installs Hugo at the version you specify. On top of that, there's a GitHub Pages Action which allows you to deploy your static content to your `gh-pages` branch or whichever branch your GitHub Pages is setup to point to.

This therefore meant that I didn't actually need to use any scripts or any custom code at all and my workflow YAML file is wonderfully simple. My code is almost exactly the same as one of the examples in the Hugo Actions repo - everything just worked - what a joy it is to say that after the 'fun' I've had debugging pipelines in certain CI tools in the past.

Here's my complete workflow to change my blog to use GitHub Actions:

1. Changed my old 'source' and 'master' branches to 'main' for the code and 'gh-pages' for the static content
1. Installed cschleiden.vscode-github-actions GitHub Actions extension in VSCode which gives you a language engine and workflow visualisation.
1. Added .github/workflows folder to the root of the repo.
1. Created a new file called 'build-site.yml'.
1. Added the new code to build my Hugo site and publish it to a gh-pages branch.
1. Pushed the code up to GitHub and watched the CI do its thing.

Here is the full workflow file with some annotations for each section:

```yaml
name: site-deployment
# Tell GH Actions to only run on a push to the 'main' branch
on:
  push:
    branches:
      - main
jobs:
  # 'deploy' is the arbitrary name of our job
  deploy:
    runs-on: ubuntu-18.04
    steps:
      # This uses the Checkout action to grab our code including the theme submodule
      - name: Git Checkout
        uses: actions/checkout@v2
        with:
          submodules: true
          fetch-depth: 0

      # Installs v0.75.1 of Hugo using the Hugo Action
      - name: Install Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.75.1'

      # Run 'hugo' to generate the static content - defaults to the ./public folder
      - name: Build Site
        run: hugo

      # This publishes our static site in ./public to the default gh-pages branch
      # GITHUB_TOKEN is an automatic token to allow authentication for Actions
      # We've also got a custom domain set up here
      - name: Deploy Site
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          cname: matthorgan.xyz

```

As you can see from the above, a super simple way to get your built and deployed on a commit. I'm looking forward to getting into the weeds with GH Actions on some more complicated projects but initial impressions are very good!
