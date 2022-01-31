+++
author = "Andreas Mosti"
date = 2021-12-16T18:01:24Z
description = ""
draft = false
slug = "deterministic-systems-with-nix"
title = "Deterministic systems with Nix"

+++


> This is a cross-post of [my contribution](https://www.bekk.christmas/post/2021/13/deterministic-systems-with-nix) to this year's [advent calendar](https://www.bekk.christmas/) we do over at Bekk. Hope you like it!

## Introduction

Setting up reliable environments for our software is tricky.
The task has kept developers and sysadmins up at night for decades. Making environments and packages truly reproducible and reliable for more than a few weeks before regression sets in, is surely no easy task. In this post, we'll see how we can set up truly deterministic, reproducible and even ephemeral environments with the help of a clever set of tools called Nix, so we can sleep better, knowing our systems can be installed from literary scratch and be guaranteed the same binary packages down to the lowest dependencies.

## What's in a name?

Let's face it, "Nix" has a quite ambiguous name that can reference a lot of things, so first, let's get the naming out of the way.

When people hear "Nix", they might think about "*nix", or the commonly spoken variant (without the asterix) "nix", the short name the industry has adopted for systems based on good old UNIX. Linux is "nix", macOS is "nix", BSD is "nix" - and in a way, "Nix" is also... well, "nix." Confused? Yeah.

Nix, in our context, refers to three things: the [Nix Expression Language](https://nixos.wiki/wiki/Nix_Expression_Language), a pure, lazy, functional language. This language makes up the foundational building blocks of the Nix package manager, which can be installed on any "*nix" system ([like Linux or macOS](https://nixos.org/manual/nix/stable/quick-start.html)) or as it's own unique Linux distro, [NixOS](https://nixos.org/). So a language, a package manager and even a distro. What's this all about?


## What makes Nix so special?
 
With the naming out of the way, what makes Nix so special? What does it have to offer that `apt`, `yum`, or `brew` don't have?

First off, it's cross platform. The Nix Package Manager [can run on the most common Linux systems, as well as macOS](https://nixos.org/manual/nix/unstable/installation/supported-platforms.html), but that is true for many package managers these days, and is not it's main advantage.

What makes Nix special is how it manages packages and dependencies. Nix guarantees reproducible packages, which means that all steps involved in building a package can be run again and again with the same outcome, and if any of the variables in the dependency chain change (all the way down to low-level packages like `libc`), it will result in a new version of this package, that can be installed side-by-side with the old version. This is possible thanks to the nature of the functional Nix language. From the docs:

>Nix is a purely functional package manager. This means that it treats packages like values in purely functional programming languages such as Haskell — they are built by functions that don’t have side-effects, and they never change after they have been built. 

Nix stores packages in the Nix store, usually the directory `/nix/store`, where each package has its own unique subdirectory such as

```sh
/nix/store/b6gvzjyb2pg0kjfwrjmg1vfhh54ad73z-firefox-33.1
```
 where `b6gvzjyb2pg0…` is a unique identifier for the package that captures all its dependencies (it’s a cryptographic hash of the package’s build dependency graph). This enables many powerful features.

In more practical terms, this is accomplished by generating hash values of all dependencies going _in_ to the package build, as well as the _outcome_ of the build itself.

## Kicking the tires

Let's look at an example package called `hello`. 
The Nix-script responsible for building the package can be found in the [nixpkgs-github repo](https://github.com/NixOS/nixpkgs)
(all packages are essentially built and installed from these scripts) and it looks like this:

<script src="https://gist.github.com/andmos/9c56554310a6a1dd653d997bcfeae943.js"></script>

We begin by installing it to the user's environment:

<script src="https://gist.github.com/andmos/c1d48189a5ad662c59bbf25c54f9bb53.js"></script>

As we can see, a lot of things was required for a simple program that prints out `Hello, World!`.

One might look at this list and think "Hey, i see curl on there - curl is already installed on my machine, why do I need it again, and won't multiple versions of the same package wreck-havoc on my machine?"

To address the first comment, this neat little trick is what makes Nix-packages self-contained and immune to what else might be installed on the system.
Other package managers, like `apt` or `brew` are often heavily dependent on there existing _one_ version of a package or it's transitive dependency.
This is why installing packages on different systems can lead to quite different results, and why a package update can break a system.
With Nix, all package dependencies comes bundled and are stored in their own hashed directories.
The model Nix follows is that every transitive dependency must be defined in a Nix-expression that can be built itself, thus supplying a dependency chain of "build instructions" all the way to the lowest parts.
If one lower-level dependency change, the main package can not be seen as the same exact version as we had before, and will be installed side-by-side with the old version, completely isolated.
This nifty feature is why Nix and NixOS has become a favorite among developers and system administrators alike, it makes for highly robust and deterministic systems, easy to update or rollback.

To address the concern of multiple versions of `curl`, let's take a look at what we have on our `PATH` after the install of `hello`:

```sh
$ which curl
/usr/bin/curl
```

`curl` being a transitive dependency for `hello` does not place it on our `PATH`, the version of `curl` provided by macOS is still in place.

If we install `curl` as a top level package, the story would be different:

<script src="https://gist.github.com/andmos/19dd36c37fa4b3afa2c942bb5a9e8f5b.js"></script>

To keep score of what version of a Nix-package is currently being used, Nix takes leverage of symlinks:

<script src="https://gist.github.com/andmos/0e5c437602621d098c0dcb7c62a06602.js"></script>

If we regret installing `curl` via Nix or something broke, Nix keeps track of the users environment in
[Generations](https://nixos.wiki/wiki/NixOS#Generations), making it easy to rollback the system:

<script src="https://gist.github.com/andmos/0361e12c6b59dd874450370052556350.js"></script>

## Creating reproducible development environments with nix-shell

Another powerful tool provided with Nix, is `nix-shell`. Software teams have always been struggling with the famous "works on my machine" syndrome, where a build or piece of code works as expected on one machine, but not on another.
Creating reproducible development environments has been the holy grail for many, and tools like [Packer](https://www.packer.io/) and [Vagrant](https://www.vagrantup.com/) takes the virtual machine way to solve this, by building VM images that can have tool pre-installed or installed via provisioning systems like [Ansible](https://www.vagrantup.com/docs/provisioning/ansible).

Another way to solve this is with containers, typically with [Docker](https://www.docker.com/) and [Docker-Compose](https://docs.docker.com/compose/).
Both virtual machines and container technology has pros and cons, but the mayor drawback is that it is quite hard to make truly reproducible environments.
Both are great for freezing a setup in time (like a VM image or a container image), but are hardly deterministic. 
A badly written `Dockerfile` can produce different results when built on two different systems.
>As a side note, it is possible to build reproducible and small Docker images [with Nix](https://nix.dev/tutorials/building-and-running-docker-images).

`nix-shell` on the other hand leverages the power of Nix to build local, reproducible, isolated and ephemeral shell-environments.

Let's say we want  `python3` but don't want to install it user/system-wide. It is possible to use `nix-shell` to provide an on-demand shell with just `python3`:

<script src="https://gist.github.com/andmos/c89cfa43a073fd6c263307ac0279e7f9.js"></script>

As we can see, no `python3` package was installed on the system, but with `nix-shell` we are able to download the package with all dependencies and make it available in a local nix-shell.
When exiting the shell, no version of `python3` is available on path.

With the Nix language, it is possible to write declarations for these shells that can be shared among the development team.
Let's say the team is maintaining a Java application, deployed on Kubernetes, and want a setup that just works™ on all systems:

<script src="https://gist.github.com/andmos/d6c853be08f78def1e6241bc9470aff5.js"></script>

This script can be stored in the root of the Java-project and added to version control. When a developer wants the environment, a simple command will provide it:

<script src="https://gist.github.com/andmos/0d76eda18d21d0f502958f464fe861e4.js"></script>

As we can se, the selected packages and dependencies are all installed.

To clean up old `nix-shell` sessions, we can simply run
```sh
$ nix-collect-garbage
```

## Conclusion

This post has been a brief intro to Nix and what at can provide in terms of reproducible, isolated systems. It is possible to do so much more than just install pre-built packages. I would recommend [How Shopify Uses Nix](https://shopify.engineering/shipit-presents-how-shopify-uses-nix) for further inspiration on how to build and deliver software using Nix, as well as checking out the [home-manager](https://github.com/nix-community/home-manager) project for managing user environments.

Interested in building your first Nix package? See the excellent [nix-tutorials](https://nix-tutorial.gitlabpages.inria.fr/nix-tutorial/first-package.html) website and start hacking!