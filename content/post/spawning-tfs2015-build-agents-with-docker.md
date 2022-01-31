+++
author = "Andreas Mosti"
date = 2016-09-24T10:48:56Z
description = ""
draft = false
slug = "spawning-tfs2015-build-agents-with-docker"
title = "Spawning TFS2015 Build Agents with Docker"

+++


The Microsoft Open Source train keeps rolling fast forward with more and more projects popping up on GitHub. When we upgraded to TFS2015 late last year, the support for Linux build agents was so-so. 
A couple of weeks ago I decided to check in again. 

To my delight I discovered that the "old" [build agent had been depricated](https://github.com/Microsoft/vso-agent) and the [new one](https://github.com/Microsoft/vsts-agent) has been implementet using [dotnet core](https://www.microsoft.com/net/core#macos).
That means cross-platform awesomeness!

It is now easy to install the agent on OSX, Ubuntu, the Red Hat family and so on.
That makes TFS much more accessible for people building other stuff than .NET. TFS has become a great product over the years, so this realy neat.

This also means that Docker can be used to spawn build agents on the fly. Somethimes we don't need to have build agents standing idle hours on end. Docker is great for spawning the agent when it actually is needed. It also allows us to install the relevant build-environment for each case. If you want to build Java, we can create a build agent with Java. Or Node. Or Mono.

The following ``Dockerfile`` gives you the idea:

```
FROM java

RUN useradd -ms /bin/bash builder
WORKDIR /home/builder

RUN apt-get update && apt-get install -y libunwind8 libcurl3 libicu52 && apt-get install wget

RUN mkdir buildAgent && cd buildAgent
RUN wget https://github.com/Microsoft/vsts-agent/releases/download/v2.107.0/vsts-agent-ubuntu.14.04-x64-2.107.0.tar.gz
RUN tar xzf vsts-agent-ubuntu.14.04-x64-2.107.0.tar.gz

USER builder
CMD ./config.sh && ./run.sh
```

The `Dockerfile` itself installs the build agent and starts an interactive configuration before starting the agent. The neat part is the `FROM` keyword. In this example, I want to build some Java code, so I base the file on the Java image.

Another tip is the scriptable setup of the agent. The `config.sh` script can be triggered noninteractive:

```
docker run -it andmos/vsts-agent ./config.sh --unattended --acceptteeeula --url http://mylocaltfsserver:8080/tfs --auth Negotiate --username DOMAIN\USER_NAME --password MyPassword --pool default --agent myagent && ./run.sh
```