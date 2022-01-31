+++
author = "Andreas Mosti"
date = 2015-03-03T19:21:09Z
description = ""
draft = false
slug = "self-hosted-http-service-in-c-with-nancy-and-topshelf"
title = "Self-hosted HTTP service in C# with Nancy and TopShelf"

+++



I found myself in need of a standalone, self-hosted HTTP Service for a REST-backend at work the other day. I like my services to be flexible and easy to deploy with a low footprint. Here's the catch: At work we write in .Net and I truly hate IIS. I kinda like C#, but I don't want my webservices to be tightly locked onto the platform-specific overhead hell that is IIS. Thanks to [OWIN](http://owin.org/), [Nancy](http://nancyfx.org/) and [TopShelf](http://topshelf-project.com/) it easy to write a self-hosted HTTP service (Ruby or Node.js style!) in C# and have it run as a standalone application or as a Windows Service. Here is a super duper easy example using Nancys Self-Host and TopShelf.


###The Code
First of all we need som packages from NuGet:

		Install-Package Nancy.Hosting.Self 
		Install-Package Topshelf 
		Install-Package Topshelf.Linux
		

First the NancySelfHost class:

		using System;
		using System.Diagnostics;
		using Nancy.Hosting.Self;

		namespace AwesomeNancySelfHost
		{
			public class NancySelfHost
			{
				private NancyHost m_nancyHost;

				public void Start()
				{
					m_nancyHost = new NancyHost(new Uri("http://localhost:5000"));
					m_nancyHost.Start();
			
				}

				public void Stop()
				{
					m_nancyHost.Stop();
					Console.WriteLine("Stopped. Good bye!");
				}
			}
		}		
		
Now we need a Nancy module to describe the route for the webservice. Lets make a simple API-example and return some JSON:

		using System;
		using Nancy;

		namespace AwesomeNancySelfHost
		{
			public class ExampleNancyModule : NancyModule
			{

				public NancyModule() 
				{

					Get["/v1/feeds"] = parameters =>
					{
						var feeds = new string[] {"foo", "bar"};
						return Response.AsJson(feeds);
					};
				}
			}
		}
		
Finaly we strap the whole thing together and make a service with TopShelf: 

		using System;
		using Topshelf;

		namespace AwesomeNancySelfHost
		{
			public class Program
			{
				public static void Main()
				{
					HostFactory.Run(x => 
					{
						x.UseLinuxIfAvailable();
						x.Service<NancySelfHost>(s => 
						{
							s.ConstructUsing(name => new NancySelfHost()); 
							s.WhenStarted(tc => tc.Start()); 
							s.WhenStopped(tc => tc.Stop()); 
						});

						x.RunAsLocalSystem(); 
						x.SetDescription("Nancy-SelfHost example"); 
						x.SetDisplayName("Nancy-SelfHost Service"); 
						x.SetServiceName("Nancy-SelfHost"); 
					}); 
				}
			}
		}

Thats all the code we need! If we know run the application, a console window will show the following:

		Configuration Result:
		[Success] Name Nancy-SelfHost
		[Success] DisplayName Nancy-SelfHost Service
		[Success] Description Nancy-SelfHost example
		[Success] ServiceName Nancy-SelfHost
		Topshelf v3.1.135.0, .NET Framework v4.0.30319.17020
	    The Nancy-SelfHost service is now running, press Control+C to exit.
		
Navigate to ``http://localhost:5000/v1/feeds`` with you're favorite browser and get yourself some JSON! 

### Install the Windows Service 

Running the console application is no use for us if we want this example API to run on a Windows Server as a Windows Service. To make it so, hit up a CMD (or Powershell) window as administrator. Navigate to the projects bin/debug folder and type 
		
		AwesomeNancySelfHost.exe install
		AwesomeNancySelfHost.exe start
If you know check the ``services``snap-in (``run => mmc``) you will see the AwesomeNancySelfHost in the list of Windows Services. Again, check ``http://localhost:5000/v1/feeds`` and check out the foobar JSON!

### Bonus Round: Linux hosting
Did you notice the ``Install-Package Topshelf.Linux`` NuGet-package and the ``x.UseLinuxIfAvailable();``statement in TopShelfs Main method? It is true, both Nancy and TopShelf runs natively in Mono, so our tiny webservice will run under Linux too, making it not only standalone but cross-platform too. If we pack the whole thing together with my [Docker-image for Mono](https://github.com/andmos/Docker-Mono) deployment get realy easy and the service can scale horizontally with ease.

### Wrapping it up

Thanks to [OWIN](http://www.asp.net/web-api/overview/hosting-aspnet-web-api/use-owin-to-self-host-web-api) and [ASP.NET V-Next](http://www.asp.net/vnext) Microsoft has got its game up on the web front. We now see a decoupling between service and server, which I think is a good thing. Specific platforms and servers should not matter when writing web-services, and it seems like Microsoft finally has realized that fact. With projects like Nancy and TopShelf I no longer fear writing web APIs in .Net, making C# a language that can be used on the entire application stack, as well as cross-platform.
