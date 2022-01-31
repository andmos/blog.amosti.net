+++
author = "Andreas Mosti"
date = 2020-04-10T09:43:29Z
description = ""
draft = false
slug = "local-reverse-proxy-with-nginx-mkcert-and-docker-compose"
title = "Local reverse-proxy with Nginx, mkcert and Docker-Compose"

+++


## Good practices from the Twelve-Factor app
When developing modern web application or services, the [Twelve-factor app](https://12factor.net/port-binding) taught us that our services

>is completely self-contained and does not rely on runtime injection of a webserver into the execution environment to create a web-facing service. The web app exports HTTP as a service by binding to a port, and listening to requests coming in on that port.

What this means is that our apps written with modern frameworks (like [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/?view=aspnetcore-3.1)) should provide their own web-servers, exposing a HTTP port, and not require anything in front for hosting, like `IIS` or Apache `HTTPD`. For local development, you should be able to run the app without any third-party component requirements for hosting, and the app should be reachable on `http://localhost:5000/` as an example.

 Now hosting the app _directly_ in this way in a production setting is something you _don't want_ to do for obvious reasons, since the infrastructure-layer of the app would grow thick, and the developer must code in all sort of hardening, routing etc, not to mention the security concerns - that self-hosted HTTP server would be exposed all on it's own as an attack-vector. When running in production, a component suitable for the [reverse-proxy](https://en.wikipedia.org/wiki/Reverse_proxy) role should be responsible for binding a public-facing hostname to the app(s), as well as do HTTPS termination - the app itself should focus on what it does best, the business logic (this is why it exists in the first place), while a component like [nginx](https://www.nginx.com/) or [HAProxy](http://www.haproxy.org/) should handle hostname binding, HTTPS and load-balance incoming requests.

 Modern platforms like [Kubernetes](https://kubernetes.io/) or [OpenShift](https://www.openshift.com/) offers _routes_ that gives the app an external-reachable hostname, and load-balances the application when running on different nodes, as well as provide HTTPS termination up-front. For small solutions not needing a container orchestrator, plain old nginx in front works great.

 All modern application _should_ be hosted with SSL and HTTPS. Thanks to projects like [Let's Encrypt](https://letsencrypt.org/), trusted SSL certificates can be obtained for free, and the world is now, slowly but surely, moving to HTTPS as default. This does not mean that our application's first meeting with HTTPS should be in a staging or production environment, it should also ble possible to develop and test locally with HTTPS as default. Thanks to modern tools, running a local reverse-proxy with a valid HTTPS certificate is quite straight forward.

## Local reverse-proxy with SSL termination

 Let's say we have a single application, `MyService`, that is written with ASP NET, running with [Kestrel](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-3.1). The app has no code for HTTPS-redirects, and knows nothing about any SSL certificate or setup, it only talks HTTP on port 80. The application is destined for a life in a container orchestrator of some kind, so it has a `Dockerfile`. To be able to run the `MyService` application via HTTPS in an environment _similar but not equal to_, let's say Kubernets, we need to run it behind a reverse-proxy when testing locally. We also need some sort of SSL certificate. Earlier in the post I mentioned Let's Encrypt that offers free certificates, but to be able to leverage it, a public-facing hostname is needed. For local development, a self-signed certificate is plenty. Now the road down self-signed certificates can be quite dirty and lead to many half-working solutions and "not-trusted" warnings in the browser. One easy solution is using a great tool called [mkcert](https://github.com/FiloSottile/mkcert). `mkcert` is a simple CLI that registers a trusted CA on your machine, both in the local certificate store and in all installed browsers, and can generate certificates from this CA.

 To install `mkcert` with `brew`:

`$ brew install mkcert`

Before installing the `mkcert` CA and generating a certificate:

<script src="https://gist.github.com/andmos/7fae6b63942f0c27f65cd1fd5dc9e47d.js"></script>

Please note, this CA and certificates generated from it is for _local_ purposes only.

Next up, let's configure `nginx` to work as a reverse-proxy with SSL termination:

<script src="https://gist.github.com/andmos/76fed99e90c2370eab3abcfd316d604e.js"></script>

This configuration will tell `nginx` to listen on `localhost`, port `5000` with the generated certificate from `mkcert`. Requests to `/` is then forwarded to the app, listening on plain old HTTP on port `80`.

The whole thing is then tied together with `docker-compose`:

<script src="https://gist.github.com/andmos/b09aeb7bdef0e0d991140e199f41ea6f.js"></script>

Now run the whole thing with

```shell
$ docker-compose up
```

Navigating to `https://localhost:5000/` reveals a nice HTTPS symbol:

<img src="https://i.imgur.com/Yo2Jqgt.png" style="zoom:50%;" />

## Note

Unless `nginx` is used as reverse-proxy in the live environment, this solution will not be _excactly_ on parity with staging or production, but the mechanisms and practices should be similar. As a rule of thumb, the Twelve-Factor app [talks about the importance of  dev/prod parity](https://12factor.net/dev-prod-parity).