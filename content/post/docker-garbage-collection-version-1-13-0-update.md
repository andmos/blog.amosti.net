+++
author = "Andreas Mosti"
date = 2017-03-01T17:24:00Z
description = ""
draft = false
slug = "docker-garbage-collection-version-1-13-0-update"
title = "Docker Garbage Collection version 1.13.0 update!"

+++


Back in 2015 I blogged about how the missing garbage collection in Docker could seriously fill up the drive on the computer you
develop Docker Images on. This was a serioust problem, and the solution (then) was the [Spotify docker-gc
image](https://github.com/spotify/docker-gc) image. Now, with the 1.13 release we *finally* got support for just this in the
docker-client: `docker system prune`. 

`docker system prune` will remove all unused data - that is hanging images (images not used by any existing container) as well as
old volumes and networks. With the cleanup command in place, what suprises me is how long it took before we finally got this quite
basic functionality in the client.  