+++
author = "Andreas Mosti"
date = 2015-08-03T17:00:45Z
description = ""
draft = false
slug = "docker-garbage-collection"
title = "Docker Garbage Collection"

+++



A mayor issue you might have run into when working on Docker-images is actual disk space. Over time, images and containers can consume a great deal of your precious drive, often when you are developing the actual image and having a test-run after each `ADD` or `RUN` block. The containers then tend to multiply as frequently as rabbits.

To solve this problem and the lack of Docker "garbage collection", the good guys over at [Spotify](https://spotify.com) has created the [docker-gc](https://github.com/spotify/docker-gc) project on GitHub. This project contains a script that removes ALL containers that has been exited over an hour ago, together with their respective images. One hour might sound extreme, but if you think about it, containers that have exited and is not running are mostly (or at least should be!) throw-away containers we have no need for anymore.

### Install and run
The script can be run standalone or create a deb-package:

    $ apt-get install git devscripts debhelper
    $ git clone https://github.com/spotify/docker-gc.git
    $ cd docker-gc
    $ debuild -us -uc -b

    # install:
    $ dpkg -i ../docker-gc_0.0.3_all.deb

    $ sudo docker-gc

The script is of course best used as a cron.hourly job.

If you like to run everything in Docker (and hey, why wouldn't you) they have also made a Dockerfile for you:

    docker build -t spotify/docker-gc .
    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock spotify/docker-gc

### Exclude images

Some images are nice to actually keep on the host (maby for dev or on a production server?)
It would have been a real bummer if there were no support for exclusion of images from garbage collection, but thankfully there is:

    touch /etc/docker-gc-exclude
    echo "spotify/cassandra:latest" >> /etc/docker-gc-exclude


### Notes
If you are running a docker version above 1.6 and experience an error with the date format like so:

      $ sudo docker-gc
      date: invalid date ‘"2015-07-25 09:26:5’
Replace the related lines in the data_parse function with these lines:

    replace_t="${without_ms/T/ }"
    replace_quote="${replace_t#\"}"
    epoch=$(date_parse "${replace_quote}")
As of 03.08.2015 this fix has not been merged to master but I guess that will happen at some point soon.
