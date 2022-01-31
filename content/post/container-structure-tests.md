+++
author = "Andreas Mosti"
date = 2018-03-05T17:15:47Z
description = ""
draft = false
slug = "container-structure-tests"
title = "Container Structure Tests"

+++


A cool new project from [Google Cloud Platform](https://cloud.google.com/) is [Container Structure Tests](https://github.com/GoogleCloudPlatform/container-structure-test). When working with containers, running tasks like unit tests as a part of the the container build stage is smart - and with [Docker multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) available there is no excuse not to.

Including tests, the Dockerfile instructions itself will throw an exitcode if some of the preconditions change.

Let's say we have the step
```
COPY ReadingList.exe.config /ReadingList/bin/Release/ReadingList.exe.config
```

and it gets tampered with,

```
COPY ReadingList.exe.config.fake /ReadingList/bin/Release/ReadingList.exe.config
```
the daemon will throw an exitcode since `ReadingList.exe.config.fake` don't exist.

So far so good. But what if the destination name gets tampered with, like this?

```
COPY ReadingList.exe.config /ReadingList/bin/Release/ReadingList.exe.config.fake
```
If the file is not touched anymore in the Dockerfile before a container is run from the image, the error won't be discovered before runtime. This is the one of the cases the container structure tests is meant to catch.

The container structure tests will validate the structure of the container image and provides a good way to do regression tests on the build artifacts themselves, so we can catch errors before the image is pushed to the repository. These tests can be used to check the output of commands in an image, as well as verify metadata and contents of the filesystem.

Let's take the example over. If the container image needs to have a config file at `/ReadingList/bin/Release/`, we can write some tests for it (and more):

<script src="https://gist.github.com/andmos/d63f2da228b2da60d0eff5a7d004ebb2.js"></script>

`fileExistenceTests` can assert files in the container image, `metadataTest` can assert expected exposed ports etc.
There are also categories for file content tests, command tests, environment variable tests and more.

To run the tests on an image, use to official container:

```
docker run -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/tests gcr.io/gcp-runtimes/container-structure-test:v0.2.1 -image andmos/readinglist -test.v /tests/imageTests/readinglist_container_tests_config.yaml
```

The output looks similar to that of any other unit test Framework:

```
Using driver docker
=== RUN   TestAll
=== RUN   TestAll/File_Existence_Test:_ReadingList.exe
2018/03/05 18:08:31 Running tests for file /tests/imageTests/readinglist_container_tests_config.yaml
=== RUN   TestAll/File_Existence_Test:_ReadingList.exe.config
=== RUN   TestAll/File_Existence_Test:_ReadingList.exe.config.template
=== RUN   TestAll/File_Existence_Test:_ReadingList.exe.config.toml
=== RUN   TestAll/Metadata_Test
--- PASS: TestAll (0.78s)
    --- PASS: TestAll/File_Existence_Test:_ReadingList.exe (0.18s)
    --- PASS: TestAll/File_Existence_Test:_ReadingList.exe.config (0.18s)
    --- PASS: TestAll/File_Existence_Test:_ReadingList.exe.config.template (0.19s)
    --- PASS: TestAll/File_Existence_Test:_ReadingList.exe.config.toml (0.22s)
    --- PASS: TestAll/Metadata_Test (0.00s)
	structure_test.go:49: Total tests run: 5
PASS
```

I'm planning to implement container structure tests as a last sanity check before an image is pushed.
With [Travis](https://travis-ci.org/) this can be done with ease:

```
script:
  - docker build -t andmos/readinglist .
  - docker run -v /var/run/docker.sock:/var/run/docker.sock -v $(pwd):/tests gcr.io/gcp-runtimes/container-structure-test:v0.2.1 -image andmos/readinglist -test.v  /tests/imageTests/readinglist_container_tests_config.yaml
```

The project is rather new, but comming from Google I suspect it will get much more love in the future. It surely fills a hole regarding CI and CD in a container world. 