+++
author = "Andreas Mosti"
date = 2019-11-03T08:43:52Z
description = ""
draft = false
slug = "automate-docker-base-image-updates-with-watchtower"
title = "Automate Docker base image updates with Watchtower"

+++


If you host some simple hobby services with plain old Docker, chances are high that you have been thinking about how to automate the deployment process. If the services are small enough and you host them on your own servers or VM's, going to the PaaS cloud or introducing Kubernetes with a sophisticated CI/CD pipeline is, in most cases, total overkill.

Why invest more time in setting up the complicated hosting and scheduling platform than it took to write that 500 lines single-container web service?

Don't fear, [watchtower](https://containrrr.github.io/watchtower/) is here.

Watchtower is a single process container that runs on your system and _polls_ a container registry (private or public, like Dockerhub) at given intervals to check for new versions of the base image on the running container(s) you want to update. If it detects a new image, Watchtower stores the parameters used to start the running container, like startup argument and environment variables, pulls down the new image, stops the running container, and starts it up again, with the same parameters, but with the new image. Easy as that. This simplifies the deployment process, and the only automation needed is a CI pipeline that builds the service's container image and pushes it to the registry when code is committed. Here is an example from my simple [ReadingList](https://github.com/andmos/ReadingList) API:

In `.travis.yml`:

<script src="https://gist.github.com/andmos/4783be0dda67cd8e74d598ef92c6006b.js"></script>

On push to master, if the build and test steps are successful, the image is tagged as `latest` and pushed to DockerHub. Nothing more to it.

Now on my private [Linode](https://welcome.linode.com/) server the ReadingList API has been started _once_ with the correct env-variables, so it's running as intended. Then Watchtower comes in:

```shell
docker run -d --name watchtower  -v /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower readinglist
```

Watchtower is now running and has access to `docker.sock` to be able to start and stop container.

When a change to ReadingList is merged and new image pushed, watchtower kicks in and replaces the running container with a new, updated version:

```shell
docker logs watchtower
time="2019-10-30T13:06:39Z" level=info msg="Found new andmos/readinglist:latest image (sha256:803aa566d2dd53f3ec774406f6bd8e20cb4e926006cdc1012dc663e206fbc9dc)"
time="2019-10-30T13:06:40Z" level=info msg="Stopping /readinglist (45ceaaa2de930b22424438d1f2d078796feead127b0ab578c0ff8ac14dc8630e) with SIGTERM"
time="2019-10-30T13:06:41Z" level=info msg="Creating /readinglist"
time="2019-10-30T13:13:38Z" level=info msg="Waiting for running update to be finished..."
```

And thats it. Be mindful, this approach has limitations. When managing larger multi-container systems in production something like Kubernetes and a more thorough setup is recommended, but for the casual, single container hobby project Watchtower works just fine.
