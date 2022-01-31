+++
author = "Andreas Mosti"
date = 2015-09-03T17:02:39Z
description = ""
draft = false
slug = "build-and-publish-asciidoc-with-docker-and-nginx"
title = "Build and publish AsciiDoc with Docker and nginx"

+++


One of the teams at work is piloting the usage of [AsciiDoc](http://www.methods.co.nz/asciidoc/) for documentation of their product. AsciiDoc is a markup-format just like Gruber's [Markdown](http://daringfireball.net/projects/markdown/), but is more advanced and offers more possibilities. Since the documentation is located alongside the source code in Git (as it should be!) I created a simple build step for easy build and deploy of the documentation with my favorite tool, [Docker](https://www.docker.com/) with public images directly from the [Docker Hub](https://hub.docker.com/). 

These steps gets triggered on each build:

    if [[ -z "$(sudo docker ps | grep webserver)" ]]; then
        sudo docker run -dt --name webserver -p 80:80 -v /usr/share/nginx/html nginx
    fi

    sudo docker run -it -v /var/buildDropLocation/build/docs:/documents/ --volumes-from webserver asciidoctor/docker-asciidoctor asciidoctor -a stylesheet=dips.css -a toc-left technical_document.adoc -D /usr/share/nginx/html

In short, we check if the [nginx](http://nginx.org/)-container is running and starts it if needed. The HTML-folder is exposed.
Next we pull down and run the [asciidoctor-image](https://github.com/asciidoctor/docker-asciidoctor) from the Docker Hub and link in the docs-folder from the build. We use [TeamCity](https://www.jetbrains.com/teamcity/), so customize these variables to fit your build system. asciidoctor then runs on the files and puts the output HTML in the volume from the nginx-container. Easy as that, live documentation directly from the latest build. 