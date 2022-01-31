+++
author = "Andreas Mosti"
date = 2015-03-03T19:26:57Z
description = ""
draft = false
slug = "build-test-and-deploy-net-apps-with-vagrant-and-docker"
title = "Build, test and deploy .NET apps with Vagrant and Docker"

+++


On a resent project at work we build a cross-platform chat-application with [Xamarin](www.xamarin.com) and [SignalR](http://signalr.net/).
The SignalR-hub was to find it's home on a Linux-server, given ASP.NETs new found love for other platforms than Windows and IIS.
We had limited time and the back-end guys were developing the hub in Visual Studio. To help them make sure the code they wrote would be Mono-compatible (and easy to deploy for testing), I turned to my two favorite pieces of open source technology: [Vagrant](https://www.vagrantup.com/) and [Docker](https://www.docker.com/).

###The code
First, I wrote a simple Dockerfile based on the latest Mono-baseimage that adds the code and runs xbuild in the build-process. When the container is run without parameters, it deploys the server.

    FROM mono:latest
    ADD SignalRServer SignalRServer
    RUN xbuild /SignalRServer/SignalRServer.sln

    EXPOSE 8080

    CMD mono /SignalRServer/SignalRServer.LocalWebServer/bin/Debug/SignalRServer.LocalWebServer.exe

To automate this process, a simple script...

    # /bin/bash
    sudo docker build -t dirc/signalrhub /vagrant/
    sudo docker run -p 8080:8080 -td dirc/signalrhub

 ...before we finally spin up a VM via this Vagrantfile and `vagrant up`.

    # -*- mode: ruby -*-
    # vi: set ft=ruby :
    Vagrant::Config.run do |config|

      config.vm.box = "virtualUbuntu64"
      config.vm.box_url = "http://files.vagrantup.com/precise64.box"

      config.vm.provision :shell, :inline => "sudo apt-get update"
      config.vm.provision :shell, :inline => "sudo apt-get install curl -y"
      config.vm.provision :shell, :inline => "curl -s https://get.docker.io/ubuntu/ | sudo sh > /dev/null 2>&1"
      config.vm.provision :shell, :inline => "/vagrant/buildAndDeploySignalRHub"
      config.vm.forward_port 8080, 8080

      end

      Vagrant.configure("2") do |config|
      config.vm.provider :virtualbox do |virtualbox|
      virtualbox.customize ["modifyvm", :id, "--memory", "1024"]
      end
      end

Thats it! With a simple `vagrant up` we get an Ubuntu VM, Docker installed, the code compiled and the hub deployed, and is available at `http://localhost:8080`.
When the Hub is finished the Docker-container can easily be moved to the production server. If the Windows-guys uses code that breaks the Mono-compability, it is easily discovered.
