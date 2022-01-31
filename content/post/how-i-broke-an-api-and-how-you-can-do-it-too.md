+++
author = "Andreas Mosti"
date = 2017-06-08T16:29:00Z
description = ""
draft = false
slug = "how-i-broke-an-api-and-how-you-can-do-it-too"
title = "How I broke an API (and how you can do it too!)"

+++


> Disclaimer: The content of this post is not nearly as dramatic as the title will have it, and for a lot of developers it will sound obvious. Based on code I have seen I nonetheless think it can learn a lot of people a valid lesson as well, and hopefully help other avoid the mistakes laziness can lead to.

In november last year [I wrote about Stratos,](http://blog.amosti.net/rolling-out-web-services-with-topshelf-chocolatey-and-ansible/) a simple web API based on Nancy with the sole purpose of listing out installed Chocolatey packages on a server. With this convenient web-process installed on our servers, the team figured out that it would be nice if it also supported [plugins](https://github.com/andmos/Stratos/blob/master/doc/Plugin.md).

With plugin support we could serve other useful info from the servers via the same API host. So far so good. The story begins when one of my colleagues had to serialize some rather complex objects to JSON. Out of the box Nancy comes with it's own JSON serializer - not using [Json.NET](http://www.newtonsoft.com/json) as the rest of the world uses. This serializer did some funny things with his object, so he wanted to use Json.NET. No problem, the Nancy guys offer [Nancy.Serialization.JsonNet](https://github.com/NancyFx/Nancy.Serialization.JsonNet) as a NuGet package.
Just slide the DLL in and Nancy grabs hold of it. God stuff.

After checking in the updated code the CI build failed on the original Stratos API tests. We expect a JSON on the following format when asking for installed Chocolatey packages:

```
[
  {
    "packageName": "chocolatey",
    "version": {
      "version": {
        "major": 0,
        "minor": 10,
        "build": 6,
        "revision": 1,
        "majorRevision": 0,
        "minorRevision": 1
      },
      "specialVersion": ""
    }
  },
  {
    "packageName": "chocolatey-core.extension",
    "version": {
      "version": {
        "major": 1,
        "minor": 1,
        "build": 0,
        "revision": 0,
        "majorRevision": 0,
        "minorRevision": 0
      },
      "specialVersion": ""
    }
  }]
```

And what we got with Json.Net was

```
[
  {
    "PackageName": "chocolatey",
    "Version": "0.10.6\r"
  },
  {
    "PackageName": "chocolatey-core.extension",
    "Version": "1.1.0\r"
  }
]
```

This was quite a surprise. The two JSON serializers were acting quite differently.

The original object should be quite simple:

```
using NuGet;

namespace Stratos.Model
{
    public class NuGetPackage
    {
        public string PackageName { get; set;}
        public SemanticVersion Version { get; set; }
    }
}
```

Right of the bat this should be no problem right?
Trying to debug this a thought struck me: the only place I need `NuGet.Core` is in this object, to have a `SemanticVersion` object. [If we look at the SemanticVersion class](https://github.com/NuGet/NuGet2/blob/2.13/src/Core/SemanticVersion.cs) there are a lot of stuff there that could mess up the JSON serializer. Not owning the class ourself also prevent us from using things like DataMember attributes to control what parts of the object should be serialized. Another quite lazy choice here is to use the `SemanticVersion` object itself as a model property. There is a lot of dead weight on the object we don't need. A better choice is to wrap the parts we need in it's own object:

```
namespace Stratos.Model
{
    public class PackageVersion
    {
        public Version Version { get; set; }
        public string SpecialVersion { get; set; }

    }
}
```

And use it in the original object:

```
namespace Stratos.Model
{
	public class NuGetPackage
	{
		public string PackageName { get; set;}
		public PackageVersion Version { get; set; }
	}
}
```

To get rid of the NuGet reference, the `SemanticVersion` class got duplicated in. It is much better to own that logic ourself.

With this refactoring the JSON response looked a lot better:

```
[
  {
    "PackageName": "chocolatey",
    "Version": {
      "Version": {
        "Major": 0,
        "Minor": 10,
        "Build": 6,
        "Revision": 1,
        "MajorRevision": 0,
        "MinorRevision": 1
      },
      "SpecialVersion": ""
    }
  },
  {
    "PackageName": "chocolatey-core.extension",
    "Version": {
      "Version": {
        "Major": 1,
        "Minor": 1,
        "Build": 0,
        "Revision": 0,
        "MajorRevision": 0,
        "MinorRevision": 0
      },
      "SpecialVersion": ""
    }
  },
```

Cool, let's ship it!

An hour later, we had 25 systems that consumed this API die. What had gone wrong? The observant reader have seen it already:

`packageName` vs. `PackageName`. Lowercase, uppercase. The Nancy serializer lowercases the keys by default, while Json.Net don't.

Why didn't the testes go red you ask? Let's look at the asserts in the test:

```
var result = browser.Get("/api/chocoPackages", with =>
			{
				with.HttpRequest();			
			});

			var resultJson = result.Body.AsString().ToLower();

			Assert.Equal(HttpStatusCode.OK, result.StatusCode);
			Assert.True(resultJson.Contains("major"));
Assert.True(resultJson.Contains("minor"));
```

`ToLower()`. Yeah. The consumer of the API did *not* use `ToLower()`, obviously.

So what can we learn from this story?

* Don't take on dependencies for a single and simple usecase

I referenced `Nuget.Core` as a NuGet package to grab hold of the `SemanticVersion` class for parsing semantic version strings to a `Version` object. That usecase is so slim that just duplicating `SemanticVersion` from GitHub to my project is a much better approach - it is just stupid to take on a NuGet dependency.

* Allways unit test as far out as possible

By having unit tests that call the actual API and assert on the expected response, the chances of breaking the API minimizes substantially. With Nancy [this is no problem](https://github.com/NancyFx/Nancy/wiki/Testing-your-application). Also, *test the types*. A simple `Contains()` on the JSON response is not enough. Always deserialize the object if possible.

* You don't know what consumers have done with your API

The last and most important lesson is that if you have published your API and you have users on it, you have no idea how the client consume it. Even the smallest changes (like going from lowercase to uppsercase keys in this example) can break the consumer. That is the last thing we want. If you have put the API out there, you should respect the consumer and allways think about what effect your changes can have on them.
