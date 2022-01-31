+++
author = "Andreas Mosti"
date = 2021-02-21T10:50:17Z
description = ""
draft = false
slug = "containerize-fluentmigrator-for-effortless-db-migrations"
title = "Containerize FluentMigrator for effortless db migrations"

+++


## Continuing the containerization

Last year I wrote about how to set up [a local reverse proxy with nginx and mkcert via Docker-Compose](https://blog.amosti.net/local-reverse-proxy-with-nginx-mkcert-and-docker-compose/).
Being able to spin up a local, production-like reverse proxy to use while developing is great, but why stop there?

Sooner or later the need for a database to store the applications data will emerge, and with any data structure, the need for change - adding, updating or deleting elements of the structure is needed. In other words, the need for _migrations_.

Back in the day, a usual practice for a team depending on a database (depending on the complexity of the application and maturity of the team), was to share a database instance for development. Setting up a local database can be tricky, and up until 2017 Microsoft's [SQL Server was only available on the Windows platform](https://blogs.microsoft.com/blog/2016/03/07/announcing-sql-server-on-linux/), requiring a VM for local development for *nix users. The single, shared instance strategy is also quite limiting for teams working in parallel on tasks requiring database and / or application code changes. A developer testing a database change on a single branch can easily break the main branch when a single instance is used.

In 2017 Microsoft release SQL Server 2017 (and now 2019) with Linux support, and with it, thankfully, [Docker support](https://hub.docker.com/_/microsoft-mssql-server).

A clean instance of MS SQL 2019 can be added to a `docker-compose` setup as easily as this:

<script src="https://gist.github.com/andmos/ec3838d72ea0e6137e9798f267ee59b4.js"></script>

With `docker-compose`, every developer on a team can have their own version of the database.

## Running migrations with FluentMigrator

The next step is bootstrapping the structure of the database. For .NET, [Entity Framework](https://docs.microsoft.com/en-us/ef/) or [FluentMigrator](https://fluentmigrator.github.io/) are popular choices. Let's focus on FluentMigrator.

A common pattern for handling migrations is creating a dedicated .NET `csproj` file where the migrations live. Let's call it `MyApp.Migrations`.
After [writing some migrations](https://fluentmigrator.github.io/articles/quickstart.html?tabs=runner-in-process), the easiest way of running them is via the FluentMigrator [dotnet tool dotnet-fm](https://fluentmigrator.github.io/articles/runners/dotnet-fm.html). With `dotnet-fm`, running migrations is as easy as

```sh
dotnet-fm migrate --processor SqlServer2016 --assembly MyApp.Migrations.dll --connection "Data Source=myConnectionString"`
```

## Bootstrapping the database with Docker-Compose

Now we have a `docker-compose` containing an MS SQL instance, and we have a `csproj` file containing some database migrations. To save us from having to run the migrations manually, the process of bootstrapping the development environment can be automated by containerizing the process running the migrations. For this, we create a `Dockerfile` that compiles the migration project, grabs the `dotnet-fm` tool for running FluentMigrator and wraps it up with an `entrypoint` for running.

The `Dockerfile`:

<script src="https://gist.github.com/andmos/b33e2f07b6b1ceb8b9e6e6bfe074f5d6.js"></script>

Some things to notice here.

The FluentMigrator library (installed with NuGet) and the `dotnet-fm` tool needs to be the same version, so the `sed` command on line 7 grabs the version-string from the `csproj` file and uses it to install the correct version of `dotnet-fm` tool on line 9.

On line 12 a little shell-script called `wait-for` is cloned. [https://github.com/eficode/wait-for.git](wait-for) is used to wrap the execution of `dotnet-fm` and _wait_ for the database the become available. This is a neat trick to handle the timing issues that can occur when running via `docker-compose`, where the migrations can be executed before the database is ready.

The `Dockerfile` is multi-stage, so the compiled library, `dotnet-fm` binary and `wait-for` script is copied to a `dotnet` runtime image. The `netcat` package installed on line 17 is a runtime dependency for `wait-for`.

With the `Dockerfile` in place, the final `docker-compose.yaml` file:

<script src="https://gist.github.com/andmos/cc5d63023d68cdfad5de953fcdc22c78.js"></script>

The whole thing spins up with `docker-compose up`. When `db` is ready, the migrations will be run and bootstraps the database.
