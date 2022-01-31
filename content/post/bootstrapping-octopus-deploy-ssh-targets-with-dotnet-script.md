+++
author = "Andreas Mosti"
date = 2018-01-03T18:43:46Z
description = ""
draft = false
slug = "bootstrapping-octopus-deploy-ssh-targets-with-dotnet-script"
title = "Bootstrapping Octopus Deploy SSH targets with dotnet-script"

+++


Those who use Octopus Deploy probably know about [Calamari](https://octopus.com/docs/api-and-integration/calamari).
In short, Calamari is a console app that functions as the deployment engine on the Octopus Deploy targets machines.  
Calamari is the app that makes sure your NuGet packages, Powershell, Bash, F# scripts etc. Is being executed on the target machine.
For Windows the [Octopus Tentacle](https://octopus.com/docs/infrastructure/windows-targets) is used by the main Octopus Server to communicate with the deployment targets, eventually installing Calamari on the first deploy.

This is all good, but what about Linux targets? There is a story here as well. Traditionally Calamari uses full .NET Framework, and can be run via Mono on Linux targets.
There is no need for a Tentacle process, [plain old SSH](https://octopus.com/docs/infrastructure/ssh-targets) is used to invoke a shell script under the hood that in turn invokes Calamari.
The Mono approach works, but is kind of "old and heavy" in the new dotnet-core world, so Calamari is now also available as a [stand-alone dotnet-core app!](https://octopus.com/docs/infrastructure/ssh-targets/self-contained-calamari)
The cool thing here is that no software has to be installed at all for Calamari to run (hence self-contained) which makes is much more lightweight and easy to provision.

Alas, there are some limitations here. Calamari can't run ScriptCS / F# scripts via the self-contained variant since these projects hasn't been targeted for dotnet-core yet. This is quite a bummer since the deployment pipeline we work with contains about 20.000 lines of `.CSX` code. Quite the investment.

But, a new hope approaches with the [dotnet-script](https://github.com/filipw/dotnet-script) project. As mentioned in another one of my posts, dotnet-script provides C# scripting based on dotnet-core, with full debug support via Visual Studio Code on _all_ OS'es, with great features like referencing NuGet packages inline from the scripts. No need for that `packages.config` file. The only catch is the requirement of the dotnet-core SDK, since dotnet-script uses NuGet and does some compilation behind the scenes. Still better to have a dependency on dotnet than full Mono.

So, Linux + Calamari + dotnet-script = awesomeness down the road.
As a PoC, I wrote an example Ansible Playbook that [installs dotnet-script](https://github.com/andmos/ansible-role-dotnet-script) on the VM, before it joins the Octopus environment via the SSH target option. To make it all full-circle, The SSH-target bootstrapping script is written in dotnet-script itself, taking advantage of the [octopus.client](https://octopus.com/docs/api-and-integration/octopus.client) NuGet package and the beautiful inline NuGet support.

The script itself is as simple as:

<script src="https://gist.github.com/andmos/b82dd2771c1db5ac3ff233f4305be465.js"></script>

This script is called via the wrapper shell script:

<script src="https://gist.github.com/andmos/3e21275d1695d4faff617b79adf4bc2a.js"></script>

And at last, completing it with a thrown-together Ansible role:

<script src="https://gist.github.com/andmos/6c2d9910c8e89aa95e0331062899469f.js"></script>

When the playbook run completes, the new Linux machine is available in the Octopus environment and dotnet-script is ready to go.
As of now there are no official support in Calamari for running dotnet-script directly, but [we are working on that](https://github.com/seesharper/Calamari) and planning to check the possibilities for adding `dotnet-script` support upstream in Calamari. It makes much more sense to use a C# script runner that can be executed on all platforms directly, making it easier to have a homogeneous codebase across Windows and Linux targets.

Another work-around is to create a NuGet package with a wrapper-script to execute dotnet-script. Example:
```
NuGetScriptPackage
│   ├── contentFiles
│   │   └── csx
│   │       └── any
│   │           └── helloworld.csx       
│   └── tools
│       └── RunScript.sh
```
Where the Octopus step points to the `RunScript.sh` script in the package:
```
#! /bin/bash

dotnet script ../content/csx/any/helloworld.csx
```

But this is no good solutions since it can't handle scripts argument or cross-platform execution in any good manner.

I believe dotnet-script can be a nice way to go for Octopus Deploy in the future, if it is to take cross-platform deployment seriously.
