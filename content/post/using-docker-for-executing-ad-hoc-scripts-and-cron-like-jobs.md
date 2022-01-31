+++
author = "Andreas Mosti"
date = 2017-11-13T18:27:55Z
description = ""
draft = false
slug = "using-docker-for-executing-ad-hoc-scripts-and-cron-like-jobs"
title = "Using Docker for executing ad-hoc scripts and cron-like jobs"

+++


### Intro

People use Docker to package applications and infrastructure dependencies for easy shipment - and it shines bright while doing it.
A less talked about usecase for Docker is executing simple ad-hoc scripts and cron-type jobs, like nightly jobs on a build server. Since Docker isolates processes, we can be much more free to experiment with different languages and runtimes not only in our applications, but also our scripts without having to think about provisioning software to the target servers. As developers we should be able to use the language we want and abstract it away. The output is what matters.


### The Case

First, let's look at the case. As readers of my blog know, I work with tools crossing the line between the Linux and Windows environments. That's why my team like Ansible. For provisioning Windows servers we use the `win_chocolatey` [module](http://docs.ansible.com/ansible/latest/win_chocolatey_module.html) Ansible provides.
We pull all the Chocolatey packages directly from the central Chocolatey repository. To be able to support offline provisioning of servers (as well as letting me sleep good at night knowing we have a backup) I need to script some syncing of the Chocolatey packages we use to our local ProGet server. Sounds easy, just write up some PowerShell and call it a day right?
Nah. Too boring. Let's see if we can do something cool.


### The Code

So grabbing the package names should be rather easy, just some string searching and digging in the Ansible files.
The next step is to download the packages (including the package dependencies) and upload them to our NuGet server. Here we can use the NuGet Command Line tool directly, or Chocolatey. Both are now cross platform and works as well on NIX systems as it does on Windows. Throw in a Docker image and we can mix some scripts.

A colleague of mine is working on a cool project called  [dotnet-script](https://github.com/filipw/dotnet-script). In a nutshell, the project provides C# scripting based on dotnet core, with full debug support in Visual Studio Code (if you haven't checked it out, [you should](https://www.strathweb.com/2017/11/c-script-runner-for-net-core-2-0/)). As we all know, dotnet core means Linux support, so after a quick PR to the project it now have a Dockerfile as well. Parsing some Ansible task files is easy when C# is the weapon.

All Ansible roles installing Chocolatey packages looks like this:

```
   - name: Installing dotnet 4.6 Target Pack
     win_chocolatey: name=dotnet4.6-targetpack source="{{ ChocoFeedUrl }}"
```
So the following parser code returns all the package names:

```
#! "netcoreapp1.1"
#r "nuget:NetStandard.Library,1.6.1"

var rolesFolder = "roles/";
var packages = new HashSet<string>();

var allFiles = Directory.GetFiles(rolesFolder, "*.yaml", SearchOption.AllDirectories);

foreach(var yamlFile in allFiles)
{
    var textLines = File.ReadAllLines(yamlFile);
    foreach(var line in textLines)
    {
        if(line.Contains("win_chocolatey") && !line.ToLower().Contains("internalsoftware"))
        {
            packages.Add(FetchPackageName(line));
        }
    }
}

private string FetchPackageName(string line)
{
    var words = line.Split(' ');
    foreach(var word in words)
    {
        if(word.Contains("name="))
        {
            return word.Replace("name=", string.Empty);
        }
    }
    return string.Empty;
}

File.WriteAllLines("scripts/nugetPackages.txt", packages);
```
Writing that without a single `Console.Writeline()` for debug is a pretty nice experience.

To download the packages and upload them to the local NuGet server a simple shell script does the trick:

```
#! /bin/bash

filename="nugetPackages.txt"

while read -r line
do
    nuget install -OutputDirectory . $line -Source https://chocolatey.org/api/v2/
    echo $line
done < "$filename"

packages=$(find **/*.nupkg)

for package in $packages
do
    choco push $package --api-key "SomeAPIKeyYouCantHaveDearBlog" --Source http://internalNugetServer/nuget/ExternalSoftware/ --force
done
```

Note that we use Nuget Command Line for downloading the package, while Chocolatey puts up less of a fuzz when pushing packages (throw in the `--force` flag and it just pushes, no questions asked).

To glue it all together without having to install any dotnet tools on the Linux server, enter the power of Docker.

```
#! /bin/bash

docker run --name packages -v $(cd ../ && pwd):/scripts:z andmos/dotnet-script scripts/GeneratePackagesFile.csx
docker run --name packagesync --volumes-from=packages:z -w="/scripts/scripts" andmos/choco ./DownloadFromChocoPushToProget.sh

docker rm -v data
docker rm -v packagesync
```

Here we first use the the `andmos/dotnet-script` image to execute the parser script, digging out all Chocolatey packages from the Ansible roles. The Ansible repo is shared from the host to the container via the `-v` flag, mounting it in the workspace folder. Next, the workspace folder is mounted to the `andmos/choco` container, giving it access to the `nugetPackages.txt`. The `andmos/choco` provides access to NuGet Command Line and Chocolatey itself.

Last but not least - we clean up after ourself. When writing self contained ad-hoc Docker pipelines like this it is important to take out the thrash.

### Summary

This code runs each night via a TeamCity build, syncing packages as we provision out new Chocolatey packages. The agent running it has only Docker installed, and that is enough. Docker provides a nice abstraction when executing simple scripts like this, making it easy to try out new languages without having to install a thousand runtimes on the servers. If I want to try out F# or Python next, it is as simple as switching out an Image. It is also really cool to see traditional Windows tools running smoothly on Linux. 
