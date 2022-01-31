+++
author = "Andreas Mosti"
date = 2015-10-20T18:23:32Z
description = ""
draft = false
slug = "fun-with-travis-ci"
title = "Fun with Travis CI"

+++


Ok, I need to swallow my own words on this one. Yesterday I stated the following on twitter:

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">I was thinking of writing a blogpost about building with <a href="https://twitter.com/travisci">@travisci</a>, but it is so easy to use that I don&#39;t see the point. <a href="https://twitter.com/hashtag/greatProject?src=hash">#greatProject</a> <a href="https://twitter.com/hashtag/CI?src=hash">#CI</a></p>&mdash; Andreas Mosti (@amostii) <a href="https://twitter.com/amostii/status/656205920922390528">October 19, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

So, yeah.

Recently I have been playing a lot with [Travis CI](https://travis-ci.com/) for building my open source projects directly from [GitHub](https://github.com/). All you need to get it working is to add a simple ``.travis.yml`` file to the root folder of the project you want to build, where the steps required to build the project is described. Typically it will look something like this:

    language: csharp
    mono:
      - latest
      - 3.12.0
      - 3.10.0
    solution: solution-name.sln
    install:
      - nuget restore solution-name.sln
      - nuget install NUnit.Runners -Version 2.6.4 -OutputDirectory testrunner
    script:
      - xbuild /p:Configuration=Release solution-name.sln
      - mono ./testrunner/NUnit.Runners.2.6.4/tools/nunit-console.exe ./MyPoject.Tests/bin/Release/MyProject.Tests.dll

As my tweet stated, this is really easy to understand just from looking at the file. We select a language, pick out some runtime versions to build on, select a solution, grab some dependencies, run the build and run some tests. commit some code and the build starts. Simple and powerful, as it should be.

Personally, I prefer to add a `build.sh` file to all my project, which makes it all even easier:  


    language: csharp
    mono:
      - latest
      - 3.12.0
      - 3.10.0
    before_script:
        - chmod +x build.sh
    script:
      - ./build.sh compile
      - ./build.sh test
Normally Travis CI uses [container technology to spawn these builds fast](http://docs.travis-ci.com/user/migrating-from-legacy/#How-can-I-use-container-based-infrastructure%3F), but there is nothing wrong with using containers for yourself in the build:

    language: csharp
    sudo: required
    services:
      - docker
    before_script:
      - chmod +x buildServer
      - ./build.sh build-local
      - ./build.sh build-docker
      - ./build.sh run-docker
    script:
      - ./build.sh unit-local
      - ./build.sh integration-local

Now this solution I like a lot. Here we allow the use of [Docker](https://www.docker.com/) thanks to the `sudo: required` and the `-docker` service. Next up we run a regular compile-build both directly on the host and in a Docker-container, before starting this container. The container allows us to run integration-tests against newly built code fast, without the need to deploy it on another server, making the build more efficient and keeps complexity low. The build test the entire solution from compile to deploy, all in a few lines of code.  