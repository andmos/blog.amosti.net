+++
author = "Andreas Mosti"
date = 2016-11-20T15:35:37Z
description = ""
draft = false
slug = "rolling-out-web-services-with-topshelf-chocolatey-and-ansible"
title = "Rolling out web services with Topshelf, Chocolatey and Ansible"

+++


I have been doing a lot of automation on the Windows platform at work lately. That sentence would have been associated with pain some years ago - but things have changed. [Ansible](https://www.ansible.com/) is nothing less than the perfect provisioning tool for both Linux and Windows, providing (in my opinion) the best level of abstraction when managing systems and state. A lot of modules are there also for Windows, so you have to put minimal of effort in the details of the provisioning steps, making it easy for people who are not developers or scripting guys to catch the gist of it. The next tool that has changed the game is [Chocolatey](https://chocolatey.org/). One of the traditional advantages Linux has had  over the Windows platform has been package managers - or a standardized interface for finding and installing software. Chocolatey is is an ambitious attempt at the same thing for Windows. Scott Hanselman [wrote about it ](http://www.hanselman.com/blog/IsTheWindowsUserReadyForAptget.aspx) back in 2013, and since then the project has catched on and kept growing. At DIPS we [package our software with Chocolatey](http://tech.dips.no/2016/08/17/Deployment-av-servere.html), giving us more scriptable flexibility over the old MSI regime. Chocolatey shines on it's own, but combined with Ansible it is _pure_ magic. Ansible provides a [Chocolatey module](https://docs.ansible.com/ansible/win_chocolatey_module.html) that let's us install Chocolatey packages as a part of the provisioning. Take the following examples from our Playbooks:



``` python
 - name: Install DotNet Framework 4.6.1
   win_chocolatey: name=dotnet4.6.1
```

```python
- name: Install Octopus Tentacle Files
  win_chocolatey: name=octopusdeploy.tentacle
```

```python
- name: Install Java JDK 8
  win_chocolatey: name=jdk8
```



You catch the point. Almost looks like `apt-get` right there.

Next up I like to show how a HTTP service can be rolled out with Ansible and Chocolatey. Since we roll out our own software as Chocolatey packages in a Continuous Delivery pipeline, the need to monitor exactly which packages are deployed to a given server at _this time_ came up. To deal with it I wrote [Stratos](https://github.com/andmos/Stratos), a simple HTTP API to report what Chocolatey packages are installed on the server. A `GET` on `/api/chocoPackages` will returns some JSON:

```json
[
  {
    "packageName": "chocolatey",
    "version": {
      "version": {
        "major": 0,
        "minor": 10,
        "build": 3,
        "revision": 0,
        "majorRevision": 0,
        "minorRevision": 0
      },
      "specialVersion": ""
    }
  },
  {
    "packageName": "DotNet4.5.2",
    "version": {
      "version": {
        "major": 4,
        "minor": 5,
        "build": 2,
        "revision": 20140902,
        "majorRevision": 307,
        "minorRevision": 21350
      },
      "specialVersion": ""
    }
  },
  {
    "packageName": "DotNet4.6.1",
    "version": {
      "version": {
        "major": 4,
        "minor": 6,
        "build": 1055,
        "revision": 1,
        "majorRevision": 0,
        "minorRevision": 1
      },
      "specialVersion": ""
    }
  }
]
```

Simple, but enough to provide data for a simple dashboard.



Stratos is written with [Nancy](http://nancyfx.org/) and [Topshelf](http://topshelf-project.com/). Topshelf is great because it allows you to install Console applications as Windows Services. Together with Nancy's self-hostable package this means that the HTTP service can be deployed without IIS, which is a good thing for simple applications like this one. The main method for Stratos looks like this:



```c#
using Topshelf.Nancy;
using Topshelf;

namespace Stratos
{
	public class Program
	{
		static void Main(string[] args)
		{
			var host = HostFactory.New(x =>
			{
				x.UseLinuxIfAvailable();
				x.Service<StratosSelfHost>(s =>
				{
					s.ConstructUsing(settings => new StratosSelfHost());
					s.WhenStarted(service => service.Start());
					s.WhenStopped(service => service.Stop());
					s.WithNancyEndpoint(x, c =>
					{
						c.AddHost(port: 1337);
						c.CreateUrlReservationsOnInstall();
						c.OpenFirewallPortsOnInstall(firewallRuleName: "StratosService");
					});
				});

				x.StartAutomatically();
				x.SetServiceName("StratosService");
				x.SetDisplayName("StratosService");
				x.SetDescription("StratosService");
				x.RunAsNetworkService();

			});
			host.Run();
		}
	}
}
```

All the configuration in one method. Sweet.



The application is packaged up as a Chocolatey package, AKA NuGet with a `ChocolateyInstall.ps1` script for the installation of the service:



```powershell
Write-Host "Installing Stratos as as windows service..."

try {

    $service_name = "StratosService"
    $process_name = "StratosService"
    $serviceFileName = "Stratos.exe"

    $PSScriptRoot = Split-Path -parent $MyInvocation.MyCommand.Definition
    $packageDir = $PSScriptRoot | Split-Path;
    $srcDir = "$($PSScriptRoot)\..\bin"
    $destDir = "$srcDir"

    $service = Get-Service | Where-Object {$_.Name -eq $service_name}

    if($service){
        Stop-Service $service_name
        $service.WaitForStatus("Stopped")
        kill -processname $process_name -force -ErrorAction SilentlyContinue
        Wait-Process -Name $process_name -ErrorAction SilentlyContinue
        Write-Host "Uninstalling $service_name..."

        $fileToUninstall = Join-Path "$srcDir\" $serviceFileName
        . $fileToUninstall uninstall
    }

    $fileToInstall = Join-Path "$destDir\" $serviceFileName
    . $fileToInstall install
}
catch {
    throw $_.Exception
}
try{
    . $fileToInstall start
}
catch{
    Write-Host "$process_name was successfully installed, but could not be started. This is most likely because of a configuration error. Please check the Windows Event Log."
}
```

The next part is to deploy the service. That is the easy part thanks to Ansible:



```powershell
- name: Install Stratos Chocolatey service
  win_chocolatey: name=stratos source=http://dips-nuget/nuget/InternalSoftware state=present upgrade=True
```

The `upgrade=True` flag will make sure that any new versions of the service get's rolled out.
