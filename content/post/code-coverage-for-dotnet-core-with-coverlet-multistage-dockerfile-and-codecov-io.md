+++
author = "Andreas Mosti"
date = 2019-05-26T11:38:51Z
description = ""
draft = false
slug = "code-coverage-for-dotnet-core-with-coverlet-multistage-dockerfile-and-codecov-io"
title = "Code Coverage for dotnet core with Coverlet, multi-stage Dockerfile and codecov.io"

+++


## Enter Coverlet

The one thing I missed when moving away from full-framework and Visual Studio to VSCode and dotnet core, was simple code coverage.

Given the easy tooling `dotnet` provides, with `dotnet build`, `dotnet test` and `dotnet publish`, I looked for something that integrated nicely with these commands without adding to much complexity to the code project itself. After som googling, I stumbled over Scott Hanselman's [blogpost](https://www.hanselman.com/blog/NETCoreCodeCoverageAsAGlobalToolWithCoverlet.aspx) about a cool little project called [Coverlet](https://github.com/tonerdo/coverlet). Coverlet was just what I was looking for:

>Coverlet is a cross platform code coverage library for .NET Core, with support for line, branch and method coverage.

`coverlet` can be installed as a `dotnet tool` with

```shell
dotnet tool install --global coverlet.console
```

to make it globally available, providing it's own [CLI tool running directly at the test assemblies](https://github.com/tonerdo/coverlet#code-coverage).

The strategy I have settled on is using the `coverlet.msbuild` package that can be added to your test projects with
```shell
dotnet add package coverlet.msbuild
```

When using the `coverlet.msbuild` package, no extra setup is needed, and `coverlet` integrates directly with `dotnet test` with some extra parameters,

```
dotnet test /p:CollectCoverage=true /p:Threshold=80 /p:ThresholdType=line /p:CoverletOutputFormat=opencover
```

The clue here is `/p:CollectCoverage=true`, the parameter that enables collection of code coverage. if no other option is specified, the coverage will be reported to the console when the tests are finished running:

```shell
+-----------------+--------+--------+--------+
| Module          | Line   | Branch | Method |
+-----------------+--------+--------+--------+
| BikeshareClient | 93.2%  | 94.6%  | 85.7%  |
+-----------------+--------+--------+--------+
```

Now the other parameters specified in the example is `/p:Threshold=80` and `/p:ThresholdType=line`. So if the code coverage drops below 80%, the build breaks here, while `/p:CoverletOutputFormat=opencover` writes a report in the [opencover](https://github.com/opencover/opencover/wiki/Reports) format.

## Multi-stage Dockerfile

For most new projects, I have found myself using a simple `Dockerfile` along with some CI/CD tool like [Travis](https://travis-ci.org/), [AppVeyor](https://www.appveyor.com/) or [Azure Pipelines](https://azure.microsoft.com/nb-no/services/devops/pipelines/). This approach helps keeping the builds simple, as large `Dockerfiles` are harder to work with. The sole purpose of `Docker` is to keep things reproducible no mather the environment it builds and runs images in, so migrating from one CI provider to another is hardly any work. Building locally will always match the result on the CI system.

But, let's say build using [multi-stage Dockerfiles](https://docs.docker.com/develop/develop-images/multistage-build/). In a multi-stage build, we separate the SDK, build and test tools in one image, while copying the resulting artifacts to another image, more suitable for production runtimes. The rule is, have a small production image containing just what is needed for running your artifacts. Just one problem: How do we take care of that `coverage.opencover.xml` file? We don't what to transfer that file to the production image to grab hold of it, code coverage results don't belong in a production image.

Thankfully, `Docker` stores _layers_ that can be brought up after building the image.
Here is our example multi-stage `Dockerfile`:

<script src="https://gist.github.com/andmos/1ccfb13473a896f598cd51cccbe3fa4c.js"></script>

In short, we build, test and publish the app with the `microsoft/dotnet:2.2-sdk` base image, before copying over the binaries to the `microsoft/dotnet:2.2-aspnetcore-runtime` image.

To use `coverlet` and extract code coverage, this line does the trick:

```shell
RUN dotnet test /p:CollectCoverage=true /p:Include="[BikeDashboard*]*" /p:CoverletOutputFormat=opencover
```

Notice the `label` on line 3:

```shell
LABEL test=true
```

With the label, it is possible to look up the id of the `docker build` _layer_ containing the code coverage file, create a container from that _image layer_ and use `docker copy` to grab hold of the coverage XML. Take a look:

```shell
export id=$(docker images --filter "label=test=true" -q | head -1)
docker create --name testcontainer $id
docker cp testcontainer:/app/TestBikedashboard/coverage.opencover.xml .
```

## Wrapping it up with Travis and codecov.io

So now we have a simple build chain with a multi-stage `Dockerfile` and code coverage generation. As a last feature, the coverage report can be used by code coverage analyzers like [codecov.io](https://codecov.io/). codecov.io [integrates with Github](https://github.com/apps/codecov), and can automatically analyze incoming pull-request and break a build if coverage drops by merging the PR. Quite nifty.

Integrating codecov.io with CI systems like Travis is done with a one-liner, thanks to the provided [upload-script](https://docs.codecov.io/docs/about-the-codecov-bash-uploader). When using Travis, not even a token is required.

`.travis` example file:

<script src="https://gist.github.com/andmos/65143919934e8f5deeb02c6705f9e780.js"></script>
