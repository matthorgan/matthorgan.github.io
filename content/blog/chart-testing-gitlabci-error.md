+++
title = "Error using chart-testing in GitLab Pipeline"
description = ""
tags = [
    "gitlab",
    "automation",
    "helm",
    "chart-testing",
    "ci/cd",
    "devops"
]
date = "2021-01-12"
categories = [
    "Development",
    "Automation",
    "DevOps"
]
+++

We've recently been moving over from Jenkins to GitLab and as part of this, I was creating a validation pipeline for our Helm charts using [chart-testing](https://github.com/helm/chart-testing). When I tried to use the tool for some basic linting using `ct lint`, I kept getting the error:

`Error: Error linting charts: Error identifying charts to process: Error running process: exit status 128`.

However, when I ran `ct lint --all`, everything seemed to work OK and the charts were analysed as expected. Looking further into the documentation for `chart-testing`, if you run `--all`, it'll ignore any git functionality and just analyse the charts. Without that parameter, it'll compare with what is already in source control and only analyse the charts that have differences.

By default, Gitlab CI does a shallow clone which means there is no git history for the tool to look at. Unfortunately, the `exit status 128` that you get back from the tool didn't help pinpoint it but once you disable the shallow clone within GitLab CI, `chart-testing` will now have the git information it needs to only analyse the charts that have changed.

FYI, to disable shallow clone in GitLab CI, add an environment variable called `GIT_DEPTH` and set it to 0:

```yaml
variables:
  GIT_DEPTH: 0
```
