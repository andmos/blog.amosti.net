+++
author = "Andreas Mosti"
date = 2016-01-02T11:32:42Z
description = ""
draft = false
slug = "dumbing-down-the-build-server"
title = "Dumbing down the build server"

+++


### Motivation
A common mistake I see many do (yes, i call it *mistake* ) is a build system too tightly linked with the build server they end up choosing. We made that mistake ourself. Being a typical .NET shop, we started out with [Microsoft Team Foundation server](https://www.visualstudio.com/en-us/products/tfs-overview-vs.aspx) back in 2007. Building with TFS in those days relied heavily on workflow scripts and build templates that had to be compiled and could *only* be run on the build server. The result? We could not build our code in exactly the same way locally as we did on the build server. As long as the codebase is small this might be OK, but when you start getting integration tests deep down failing on the server and not locally you are in trouble. Another clear problem we found with the TFS approach was how people treated the build server: like a black box. Of all the developers working in the company, only 4 knew how the build system was working. The whole thing got far too complex and switching over to something like [Team City](https://www.jetbrains.com/teamcity/) would force us to make the entire build system from scratch. We tried this too - and without thinking about it ended up using the [Team City 
environment variabes](https://confluence.jetbrains.com/display/TCD9/Predefined+Build+Parameters) far too much. The result? The new build system also got tied strongly against Team City. If you ever find yourself in some of these situation, press the big red button and think.

### What is a build server? 
A build server is nothing more than a *worker*. You send it tasks to do, and it does it. Simple as that. Hell, I don't like the words **build server** or **CI server**, it makes it sound more complex than it really is. The important part is not the build server but the *build system*. With a good build system the server itself can be dumbed down to just running i cron job, nothing more. The key point here is to not use all the fancy configuration options the build server has - that should be a part of the build system itself. With this in mind - let's do it right.

### Do it right 
When we started out working with the new build system, we had some clear rules to follow: 

* The system must do *exactly*  the same things locally and on the server.
* The system should be made in an easy to understand scripting language.
* Each team owns its own build, but common code should be shared.
* Use as much [Vanilla](https://en.wikipedia.org/wiki/Vanilla_software) methods as possible.
* The system should be simple without too much lock-in to the build server we choose to use. 

Building for .NET has become much better the recent years. Where we before called MSBuild directly from PowerShell we now have [psake](https://github.com/psake/psake) inspired by Ruby's  [rake](https://github.com/ruby/rake). We have some mixed experience with Powershell so we early turned this down. If you don't mind Powershell, psake seems to be a good choice.

Next up we looked at [FAKE](https://fsharp.github.io/FAKE/), a popular buildsystem written in F#. NRK has [written about switching to FAKE](https://nrkbeta.no/2015/11/10/how-i-learned-to-stop-worrying-and-love-the-ci-server/). Some teams ended up using FAKE on some independent modules, but we ended up not using it because of the learning curve F# would bring to most of our developers. 

Thanks to Microsofts [Roslyn compiler](https://roslyn.codeplex.com/) we discovered [ScriptCS](http://scriptcs.net/). ScriptCS let's you use C# as a scripting language, offering REPL and everything else you would expect from a scripting language. Since the language is C# every developer working can read and modify the scripts, no learning needed. Perfect. ScriptCS can also reference DLLs from GAC or a binary folder. 

We ended up creating a `tools\` folder with a `build.csx` and a `common.csx` file. The `common.csx` file is centrally shared in source control and acts like our library with all methods needed for building tasks, like MSBuild, file management and calls to the NuGet API and triggers for [Octopus Deploy](https://octopus.com/). The `build.csx`  file is the build file itself specified for the solution(s) to build, using methods from `common.csx`. The build system itself is distributed as a NuGet package to be installed installed to the root of the project we want to build. A `build.bat` file wrapps the build system and can be triggered like so:
		
	./build.bat clean init compile unit integration package deploy

This flow cleans, pulls down dependencies, compiles, runs unit- and integrationtests, packages NuGet or Chocolatey packages and deploys to ProGet and Octopus Deploy. These steps are exactly alike locally and on the build server, just a command line call to build.bat. If ScriptCS is not installed on the system, the bat-file will do it for you. 

### Conclusion

It took some time, but after a while we had a build system we were very satisfied with. The shared code is constantly changing as teams finds new needs. ScriptCS has been a success and have removed a lot of the magic from the build process. The teams have taken a lot more ownership over building than they had before. By pushing logic from the build server and down to the build system itself, everything works perfectly with Team City, TFS2015 and Jenkins, just as it should be. 

### Note

If you don't want to write a ScriptCS-based build system from scratch, take a look at Cake. Cake is pretty much a ScriptCS based framework for building with lots of extension possibilities.