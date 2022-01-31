+++
author = "Andreas Mosti"
date = 2015-08-07T14:54:46Z
description = ""
draft = false
slug = "managing-nuget-dependencies-with-paket"
title = "Managing NuGet dependencies with Paket"

+++


Since we mainly write .NET code at work, the only real choice for packing dependencies and applications is [NuGet](https://www.nuget.org/) and [Chocolatey](https://chocolatey.org/), everything stored in different feeds on the excellent [ProGet](http://inedo.com/proget/overview) server.  For CI we build our code with [TFS](https://msdn.microsoft.com/en-us/vstudio/ff637362.aspx) and [Team City](https://www.jetbrains.com/teamcity/) using some hand-made build scripts written in [scriptcs](http://scriptcs.net/), while deployment is done with good old [Octopus Deploy](https://octopusdeploy.com/).

All this works perfectly in our in-house environments, everything is just a `choco install`  away.
The only catch her is that we want to ship these packages to our customers as standalone installations. To do this we have  written some custom powershell-scripts that wraps chocolatey and points to a local packages-folder where the correct .nupkg-files for the applications are stored. Since we want to use this exact installation mechanism in our CI pipeline (aka eating your own dog food) we had to be creative.

Chocolatey is great for installing software, but the easiest way to actually grab the .nupkg-files without installing them (and get the version number as a part of the .nupkg filename, chocolatey does not include this for some reason) is to use the NuGet CLI itself:

      nuget install mypackage-server -Source http://myproget -OutputDirectory "C:\SetupScript\Packages"

Here we ran in to a much debated issue: NuGet's way of managing dependencies. By default NuGet resolves the **lowest** available (and legal) version of all dependencies. I understand that this is a safe choice, but for CI we always want to test and deploy the latest version of all packages and its dependencies. The worst part is, after hours of googling and trying we found no way to force NuGet to use the highest available versions of dependencies. To solve this issue I turned to [Paket](http://fsprojects.github.io/Paket/).

Paket is a wonderful project that takes care of dependencies in your project for you in a much more elegant (and Ruby-like) way then what NuGet itself does. Here is an example:

    $ mkdir .paket
    $ wget https://github.com/fsprojects/Paket/releases/download/1.23.0/paket.exe -p .paket/
    $ touch paket.dependencies

In the `paket.dependencies` file, put in your dependencies like this:

    source https://nuget.org/api/v2

    nuget Castle.Windsor-log4net >= 3.2
    nuget NUnit

    github forki/FsUnit FsUnit.fs

To install dependencies:

    $ .paket/paket.exe install

All dependencies (including correct, transitive dependencies in **latest** versions!) are stored in the `paket.lock` file:

    NUGET
      remote: https://nuget.org/api/v2
      specs:
        Castle.Core (3.3.3)
        Castle.Core-log4net (3.3.3)
          Castle.Core (>= 3.3.3)
          log4net (1.2.10)
        Castle.LoggingFacility (3.3.0)
          Castle.Core (>= 3.3.0)
          Castle.Windsor (>= 3.3.0)
        Castle.Windsor (3.3.0)
          Castle.Core (>= 3.3.0)
        Castle.Windsor-log4net (3.3.0)
          Castle.Core-log4net (>= 3.3.0)
          Castle.LoggingFacility (>= 3.3.0)
        log4net (1.2.10)
        NUnit (2.6.4)
    GITHUB
      remote: forki/FsUnit
      specs:
        FsUnit.fs (81d27fd09575a32c4ed52eadb2eeac5f365b8348)

The files end up in a folder called `Packages` with .npkg files and all, just how we like it.

As a bonus, here is the powershell-script Octopus Deploys runs:

    $SetupRoot = "C:\SetupScript"
    $SetupPs1 = "SetupRoot\Setup.ps1"
    $OFS = "`r`n"

    Write-Host "Clearing old cache..."
    Remove-Item SetupRoot\packages\* -Recurse -Force -ExcludeSetup.psm1,Setup

    if (-not $packageName) {
        throw "Please specify the name of a package to install."
    }

    if($version){
        $versionString = "-Version $version"
    }

    Write-Host "Fetching packages..."

    cd $SetupRoot

    Set-Content -Value "source $sourceFeed $OFS" -Path $SetupRoot\paket.dependencies
    Add-Content -Value "nuget $packageName" -Path $SetupRoot\paket.dependencies


    if(Test-Path $SetupRoot\paket.lock){
        Write-Host "Removing paket.lock file"
        Remove-Item $DIPSSetupRoot\paket.lock
    }

    & .paket/paket.exe install

    Write-Host "Discovering chocolatey packages..."
    Copy-Item $SetupRoot\packages\*\*.nupkg $SetupRoot\packages

    Write-Host "Installing chocolatey package..."
    if ($myval -eq $null) { "new value" } else { $myval }

    if (-not $version){
        $version = [String]::Join(".",(Get-ChildItem $SetupRoot\packages\$packageName*.nupkg)[0].Name.Split('.'),1,4)
    }

    $Config = Get-Item "$SetupRoot\Packages.config"
    $ConfigContent = [xml]@"
    <?xml version="1.0" encoding="utf-8"?>
    <packages>
      <package id="$packageName" version="$version" />
    </packages>
    "@
    $ConfigContent.Save($Config)

    & $SetupRoot\Setup.ps1
