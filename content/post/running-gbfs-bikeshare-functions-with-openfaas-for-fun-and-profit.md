+++
author = "Andreas Mosti"
date = 2019-06-04T18:22:59Z
description = ""
draft = false
slug = "running-gbfs-bikeshare-functions-with-openfaas-for-fun-and-profit"
title = "Running GBFS bikeshare functions with OpenFaaS for fun and profit"

+++


## Intro

Micro-mobility has gotten a lot of hype over the last couple of years. In many cities all over the world rentable city bikes, cargo bikes and electrical scooters has popped out and seriously changed the way people move, and it doesn't look like you can [spell "smart city" without micro-mobility](https://companies.bnpparibasfortis.be/en/article?n=will-micro-mobility-redesign-the-smart-city). The consulting company McKinsey is estimating that the value of the micro-mobility market will reach a [value of $200 billion to $300 billion in the United States in 2030](https://www.mckinsey.com/industries/automotive-and-assembly/our-insights/micromobilitys-15000-mile-checkup). What gives micro-mobility systems advantage is the accessability and digital-first approach providers have taken. Most bikeshare systems are activated via smartphones, and the bikes themselves are IoT devices producing data [for the providers](https://urbansharing.com/) or [the public](https://trondheimbysykkel.no/en/open-data) to explore. To make micro-mobility more useful interoperability is important, and many standards have surfaced. One of these is the [General Bikeshare Feed Specification](https://github.com/NABSA/gbfs) (GBFS).

## General Bikeshare Feed Specification (GBFS) and use cases

GBFS is an open data standard for bikeshare systems developed by [North American Bikeshare Association](http://www.nabsa.net). GBFS provides info about the system itself, including real-time data like available stations, capacity, available bikes and locks etc. This API can be freely used to build 3rd party systems or apps. [How about a dashboard that shows your closest station and local weather forecast](https://github.com/andmos/BikeDashboard), or [Amazon Echo skill to check for available bikes](https://github.com/gffny/blue-bike-skill)?
The [GBFS systems overiew](https://github.com/NABSA/gbfs/blob/master/systems.csv) currently contains 228 providers using GBFS.

A search for GBFS [on github topics](https://github.com/topics/gbfs) shows a lot of project integrating with the standard, including my [GBFS Bikeshare client for dotnet](https://github.com/andmos/BikeshareClient). This client is a great starting point for exploring several exiting concepts: serverless and function as a service.

## Serverless, function as a service and OpenFaaS

[Serverless architecture](https://martinfowler.com/articles/serverless.html) comes as a result of the rising popularity of cloud computing, where providers like Google, Microsoft and Amazon have raised the abstraction level when deploying software. At the infrastructure as a service (IaaS) level you have to mange VMs, the platform as a service (PaaS) level want your binaries or containers, while the function as a service (FaaS) provider needs one thing: your code. The runtime, scalability etc. Is taken care of by the cloud vendor.

FaaS can be looked at as breaking up the [microservice pattern](https://martinfowler.com/articles/microservices.html) into smaller pieces. Examples of functions can be transforming input data and store it in a database, resize images, handle messages on a queue or, to stay in the micro-mobility domain, check the availability status on a bikeshare station.

The biggest criticism directed at serverless and FaaS is vendor lock-in. Amazon has Lambda, Microsoft has Azure Functions, and Google has Cloud Functions. Since these platforms require plain code to run, some platform specific boilerplate or project types are needed for each platform, thus not contributing to portability between vendors.

So vendor lock-in is one thing, but I would also like the possibility to run a serverless FaaS solution on that old VMWare cluster in the basement, the Mac mini rack or on the Raspberry Pi spotted in the wild. Luckily, [OpenFaaS](https://www.openfaas.com/) is here to help. OpenFaaS is a framework that leverages container technology to run functions in containers on top of orchestrators like Docker Swarm and Kubernetes. This breaks the FaaS architecture free from the cloud vendors, providing the freedom to deploy serverless application in any environment offering a container orchestrator.

![OpenFaaS architecture](https://pbs.twimg.com/media/DFrkF4NXoAAJwN2.jpg)
The OpenFaaS architecture.

## Baby's first GBFS bikeshare functions

So let's put OpenFaaS to work and build some GBFS powered bikeshare functions.

For development purposes, running OpenFaaS via Docker Swarm is a good approach.
[The installation of OpenFaaS and initialization of a Swarm cluster is straight forward](https://docs.openfaas.com/deployment/docker-swarm/).

My preferred language is `C#` and `dotnet core`, so to write the `dotnet` function a [template](https://docs.openfaas.com/cli/templates/#templates) is needed.
Github user [burtonr](https://github.com/burtonr) has written a nice [dotnet template](https://github.com/burtonr/csharp-kestrel-template) with Kestrel and async support, perfect for high performance HTTP functions. To fetch the template:

```shell
$ faas-cli template pull https://github.com/burtonr/csharp-kestrel-template
```

Next, let's have a look at what a typical function might look like. The [GBFS systems list](https://github.com/NABSA/gbfs/blob/master/systems.csv) is currently in `CSV` format, but I prefer to abstract it away and offer it as `JSON` via HTTP endpoint.

To create a new OpenFaaS function:

```shell
$ faas-cli new --lang csharp-kestrel gbfs-systems-function
Function created in folder: gbfs-systems-function
Stack file written: gbfs-systems-function.yml
```

As the output shows, a folder for the function and `stack` file are now created, next to the template.

```shell
$ tree
.
├── gbfs-systems-function
│   ├── FunctionHandler.cs
│   └── FunctionHandler.csproj
├── gbfs-systems-function.yml
└── template
    └── csharp-kestrel
        ├── Dockerfile
        ├── Program.cs
        ├── Startup.cs
        ├── function
        │   ├── FunctionHandler.cs
        │   └── FunctionHandler.csproj
        ├── root.csproj
        └── template.yml
```

To write the function, `gbfs-systems-function/FunctionHandler.cs` is edited. Here is the full function:

<script src="https://gist.github.com/andmos/13822f83e76cfab2b764d40f6ca884e8.js"></script>

The `Handle` method is entrypoint for the function. In this case the input is not validated, but upon triggering the function parses the `systems.csv` file and serializes it as JSON.

To build and deploy the function, take a look at the `stack` file:

<script src="https://gist.github.com/andmos/5f8244497cf2bf36687d14d8ffdb3e7d.js"></script>

Notice the `image:` tag. Since OpenFaaS runs functions in Docker containers, this is the name of the container image that is created, and is the artifact of the build.

To build, deploy and trigger the function:

```shell
$ faas-cli build -f gbfs-systems-function.yml
$ faas-cli deploy -f gbfs-systems-function.yml

$ echo "" |faas-cli invoke gbfs-systems-function # or via Curl
$ curl -d "" localhost:8080/function/gbfs-systems-function
```

The two last commands will return `JSON` straight from the new function.

To push the image to [Docker hub](https://hub.docker.com/):

```shell
$ faas-cli push -f gbfs-systems-function.yml
```

## Building a GBFS powered Slack bot

So that example function was rather simple. Let's make something more useful: A Slack bot for showing available bikes and locks at bikeshare stations. Right off the bat this seems like a nice use case for multiple functions, since a function should optimally do only one thing. So for the bot we need a `bikeshare-function` that takes the name of a bikeshare systems station as input, and return the number of available bikes and locks for this station as output.

Not much code needed here neither:

<script src="https://gist.github.com/andmos/a8116f0121bd8e75d4d3371a01642775.js"></script>

To link the function ta a GBFS system, the GBFS discovery URL must be exposed to the function via environment variable in the `stack` file.

When deployed, the `bikeshare-function` can be invoked:

```shell
$ curl -d "skansen" localhost:8080/function/bikeshare-function
{"Name":"Skansen","BikesAvailable":19,"LocksAvailable":2}
```

The next function is the Slack bot itself. The bot will trigger on mentions and call the `bikeshare-function` with a bikeshare station name. For creating Slack apps, [see the documentation](https://api.slack.com/slack-apps).

As expected, under 100 lines of code here too:

<script src="https://gist.github.com/andmos/6e992d653377f7bb934ddb817cf47388.js"></script>

Now there are a couple of things to notice here.
For the bot to be able to call the `bikeshare-function`, it needs to know where the [OpenFaaS API gateway](https://docs.openfaas.com/architecture/gateway/) is. When running via Docker Swarm this address defaults to `http://gateway:8080/`, but it is [recommended to make this address customizable](https://github.com/openfaas/workshop/blob/master/lab4.md#call-one-function-from-another), as it may vary from environment to environment and other orchestrators.

The next thing to notice is secrets. The bot needs a OAUTH token when connecting to Slack, and this token should be kept secret. OpenFaaS [integrates with Swarm and Kubernetes secrets](https://docs.openfaas.com/reference/secrets/), both reachable from `openfaas-cli`:

```shell
$ faas-cli secret create secret-api-key \
  --from-file=slackToken.txt
```

The token is then written to a file inside the container that must be read up:
```csharp
var botToken = File.ReadAllText(@"/var/openfaas/secrets/bikeBotSlackToken");
```

The final `stack` file:

<script src="https://gist.github.com/andmos/b7037ab2266393737db10097366bc20f.js"></script>


To initialize the Slack bot:
```shell
$ curl -d "" localhost:8080/function/bikeshare-slack-function
Bot initializing
```

The bot is now online and can be asked for station status:

<img width="651" alt="Skjermbilde 2019-06-04 kl  22 00 29" src="https://user-images.githubusercontent.com/1283556/58909797-4d732c00-8714-11e9-8bf6-026fe7e1dff5.png">

## Conclusion

It is exiting times for micro-mobility and the city of the future. For solutions like bikeshare systems to reach it's potential and help cities become more accessible, integration with other smart city systems is almost a requirement. Thanks to open standards like GBFS and the serverless paradigm, creating new application leveraging and combining data is only a couple of code lines away. Frameworks like OpenFaaS helps democratize serverless and FaaS, giving developers tools to run functions where and how they want.

Finally, all code for this post [can be found on Github.](https://github.com/andmos/BikeshareFunction).
