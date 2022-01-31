+++
author = "Andreas Mosti"
date = 2015-10-31T15:01:59Z
description = ""
draft = false
slug = "trondheim-developer-conference"
title = "Trondheim Developer Conference 2015"

+++



This year I had the pleasure of being one of the talkers at the fourth annual [TDC](http://2015.trondheimdc.no/) in Trondhiem.
The lineup included big international tech-names like [Scott Hanselman](http://www.hanselman.com/), [Sahil Malik](http://blah.winsmarts.com/), [Seb Lee-Delisle](http://seb.ly/) and [Scott Allen](http://odetocode.com/about/scott-allen), and I had a lot of fun making my debut as a conference speaker along side these people.

The talk i brought was **Simple crossplatform REST-Service with .NET, Vagrant and Docker**, a walkthrough on how to make crossplatform server components in .NET with [Vagrant](https://www.vagrantup.com/) and [Docker](https://www.docker.com/) as key tools, helping us focus on integration testing and production-like deployment from the first lines of code written. This is also a subject I have [blogged](http://blog.amosti.net/build-test-and-deploy-net-apps-with-vagrant-and-docker/) some about before. The essence of the talk is that .NET and C# now is all you need to know to write the entire stack of your applications, including the mobile client code for iOS and Android via [Xamarin](https://xamarin.com/) to the server side part with [Mono](http://www.mono-project.com/), letting you choose what platform to run on.

![](http://i.imgur.com/bJeyynv.jpg)

I gave some tips on frameworks and libraries to use when writing a simple REST-Service, including [NancyFX](http://nancyfx.org/), [Topshelf](http://topshelf-project.com/) and [Dapper](https://github.com/StackExchange/dapper-dot-net).

My session will be released by the TDC people shortly, until then - here are the slides.

<script async class="speakerdeck-embed" data-id="3191aeafb0bf493b8be90abe01639bce" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

EDIT: Here is also the talk (in norwegian): 

<iframe src="https://player.vimeo.com/video/144964559" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>


After the talk I walked back stage and helped Scott Hanselman out with his upcoming [ASP.NET 5](http://www.asp.net/vnext) talk, where he showed me the future of open source Microsoft, running on all platforms without anything installed thanks to the [CoreCLR](https://github.com/dotnet/coreclr). He told me he planned on asking some random people in the audience for a Mac to show how easy it was to run a sample ASP.NET 5 app with the CoreCLR, so I placed my co-worker [@hjerpbakk](http://hjerpbakk.com/) on the front raw. Sure enough, he had to take the stage with his Mac.

![](http://i.imgur.com/6Ba2BF7.jpg)

The tricky bit proved to not be getting asp.net to run on a Mac, but tackling the Norwegian keyboard. Hanselman joked as always, and his sessions ended up, not surprisingly, to be some of the best the entire conference had to offer.   