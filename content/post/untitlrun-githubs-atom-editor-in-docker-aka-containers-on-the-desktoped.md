+++
author = "Andreas Mosti"
date = 2015-05-23T13:09:31Z
description = ""
draft = false
slug = "untitlrun-githubs-atom-editor-in-docker-aka-containers-on-the-desktoped"
title = "Run Github's Atom editor in Docker (Aka. Containers on the desktop)"

+++



By now everybody loves [Docker](https://www.docker.com/). I mean, what is there not to love? Never have it been easier to program, pack and deploy you're applications and getting out of [dependency hell](https://www.youtube.com/watch?v=3N3n9FzebAA) while keeping them isolated from each other.
Docker has solved the whole "download and install this on the server, but remember to have the right version of Java (no, not that one!) and Tomcat (7, not 8)" - problem. All servers are happy.

A couple of months ago I read a [blogpost](https://blog.jessfraz.com/post/docker-containers-on-the-desktop/) by the talented Docker engineer [Jessie Frazelle](https://github.com/jfrazelle) who has made a large collection of Docker images for the [desktop](https://github.com/jfrazelle/dockerfiles).
The FREAKING desktop. To [quote](https://twitter.com/frazelledazzell/status/596345044912635904) Jessie:

<blockquote class="twitter-tweet" lang="no"><p lang="en" dir="ltr">The worst thing someone could do if I ever left my computer unlocked (which will never happen) is install something directly on my host</p>&mdash; jessie frazelle (@frazelledazzell) <a href="https://twitter.com/frazelledazzell/status/596345044912635904">7. mai 2015</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Feeling inspired by this crazy lady I decided to try it out myself. So here is a Dockerfile for the [Atom](https://atom.io/) editor you can run on you're Linux desktop (because, why the hell not?)

### Dockerfile


    # Based on work by Jessie Frazelle, https://github.com/jfrazelle/dockerfiles/blob/master/atom/Dockerfile
    FROM ubuntu:14.04
    MAINTAINER Andreas Mosti <andreas.mosti@gmail.com>

    RUN apt-get update && apt-get install -y \
        build-essential \
        ca-certificates \
        curl \
        git \
        libasound2 \
        libgconf-2-4 \
        libgnome-keyring-dev \
        libgtk2.0-0 \
        libnss3 \
        libxtst6 \
        --no-install-recommends

    RUN curl -sL https://deb.nodesource.com/setup | bash -
    RUN apt-get install -y nodejs

    RUN git clone https://github.com/atom/atom /src
    WORKDIR /src
    RUN git fetch && git checkout $(git describe --tags `git rev-list --tags --max-count=1`)
    RUN script/build && script/grunt install

    # set up user and permission
    RUN export uid=1000 gid=1000 && \
        mkdir -p /home/developer && \
        echo "developer:x:${uid}:${gid}:Developer,,,:/home/developer:/bin/bash" >> /etc/passwd && \
        echo "developer:x:${uid}:" >> /etc/group && \
        echo "developer ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/developer && \
        chmod 0440 /etc/sudoers.d/developer && \
        chown ${uid}:${gid} -R /home/developer

    USER developer
    ENV HOME /home/developer
    
    # get my configfiles from github
	RUN mkdir .atom &&  \ 
    git clone https://github.com/andmos/dotfiles.git && \ 
    cd dotfiles/atom; ./configureAtom

    CMD /usr/local/bin/atom --foreground --log-file /var/log/atom.log && tail -f /var/log/atom.log


### Build and run

    wget https://raw.githubusercontent.com/andmos/Docker-Atom/master/Dockerfile
    docker build -t atom
    docker run -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=$DISPLAY atom

Another cool thing we can do is to attach a [data container](https://docs.docker.com/userguide/dockervolumes/) with the
``--volumes-from=`` flag. Here is how to build a data container from a git-repo and attach: 

    docker run --name coffee -e repo=https://github.com/andmos/coffee.git -t andmos/git
    docker run --volumes-from:coffee -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=$DISPLAY atom
open ``/var/workspace/coffee`` and there is the repo! 

This image has lots of possibilities - happy hacking!
