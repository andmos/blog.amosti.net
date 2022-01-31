+++
author = "Andreas Mosti"
date = 2019-10-03T14:59:13Z
description = ""
draft = false
slug = "github-actions-and-publishing-artifacts-to-azure-blob-storage"
title = "Github Actions and publishing artifacts to Azure Blob Storage"

+++


### Intro

[Github Actions](https://github.com/features/actions) is a welcomed edition to the (still) growing world of CI/CD tools.
Since Actions is Github's own tool, it integrates more closely to your repo and the Github Workflow, with actions to automate tasks around issues, pull-requests, releases etc. Writing a task that regularly, say, check issues and mark them as stalled if it hasn't been any activity for some time has, would mean leveraging the Github API when running in other tools, while abstractions for these kinds of integrations are present directly in Github Actions. That makes their "workflow" semantics more comprehensive than just plain CI/CD capabilities.

Extendability is at the core of Github Actions. All workflows consists of one to many actions, and these actions can be run natively or via containers. Referencing a third party actions is as easy as knowing the action's github namespace.

<script src="https://gist.github.com/andmos/22a0276f9288c9eb281fc49e6833a114.js"></script>

In this example the _workflow_ `Tests` will trigger on `git push`, run on macOS, Ubuntu and Windows, checkout code with the action [actions/checkout](https://github.com/actions/checkout), and install Python via [actions/setup-python](https:(//github.com/actions/setup-python). These two actions are "official", hence the "actions" namespace. The next task installs the Python package manager [Poetry](https://poetry.eustace.io/) and is a third party action: [dschep/install-poetry-action](https://github.com/dschep/install-poetry-action). Since these actions directly reference living repositories, specifing a release or branch (`dschep/install-poetry-action@v1.2`) will save you from some unpleasant discoveries.

To speed up the build, these actions can be run directly from DockerHub:

<script src="https://gist.github.com/andmos/1ddb8949fba768fc6373c91beab4f7a1.js"></script>



### Uscase: Upload artifacts to Azure Blob Storage

Github Actions is still in it's early days, so there are not actions for everything just yet. The other day I needed a workflow to build and publish an Electron App to MacOS and Linux, with the artifacts stored in Azure Blob Storage. In Azure Blob Storage we have two containers, one for `dev` and one for `release`, so the app can be tested before released out to the world. Here is the workflow:

<script src="https://gist.github.com/andmos/416601771109493b49aba3591e3f7c2c.js"></script>

So this workflow will only trigger on push to the `dev` branch. 
[actions/setup-node@master](https://github.com/actions/setup-node) installs node on version `12.10`, and `electron-builder` is used to package and sign (the macOS) app. Here we also see secrets in play, `${{ secrets.BASE_64_CERT }}` holds an encryptet
signing certificate.

Finally, I could not find a suitable action for uploading to Azure Blob Storage directly, but going via [azure/actions/login](https://github.com/azure/actions/) worked great to auth against Azure and give access to the `az` CLI. All that is needed is generating [Azure Service Principal](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest) to give the workflow access to just the Blob Storage Containers, store them as a secret and we are good to go. For a great intro to using dotnet core with Github Actions, check out [Runar's post](https://hjerpbakk.com/blog/2019/10/03/asp-net-core-and-github-actions).
