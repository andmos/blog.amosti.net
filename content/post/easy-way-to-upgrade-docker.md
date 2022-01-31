+++
author = "Andreas Mosti"
date = 2015-05-04T18:20:21Z
description = ""
draft = false
slug = "easy-way-to-upgrade-docker"
title = "Easy way to upgrade Docker"

+++


If you use a system with a strickt package repository (like Debian stable, CentOS, RedHat etc.) chances are new releases of Docker won't be pushed right after release. A sweet trick I use on my CentOS machines is to just wget down the new binary like so: 

		service docker stop
        wget https://get.docker.com/builds/Linux/x86_64/docker-latest -O /usr/bin/docker
        service docker start
        
Since Docker has released with quite a steady pace (and I like latest and greatest) I have eaven made an alias for the job - 

		alias getLatestDockerBinary='sudo wget https://get.docker.com/builds/Linux/x86_64/docker-latest -O /usr/bin/docker'
        
Hope this might help somebody save time in the future.