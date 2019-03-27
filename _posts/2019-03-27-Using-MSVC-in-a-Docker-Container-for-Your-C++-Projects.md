---
title: "Using MSVC in a Docker Container for Your C++ Projects"
layout: post
comments: true
date: 2019-03-27 17:23
image:
headerImage:
tag:
star: false
category: IT trend
author: jihun
---

<!--more-->

# Overview

Containers encapsulate the runtime environment of an application: the file system, environment settings, and virtualized OS are bundled into a package. [Docker containers](https://www.docker.com/what-container) have changed the way we think about build and test environments since they were introduced five years ago.

Visual Studio’s setup and install expert, Heath Stewart, blogs regularly about how to install the [Visual Studio Build Tools](https://aka.ms/buildtools) in a Windows Docker Container. Recently he explained [why you won’t find a container image for build tools](https://blogs.msdn.microsoft.com/heaths/2018/06/14/no-container-image-for-build-tools-for-visual-studio-2017/). Basically, any install that has a workload installed that you aren’t using is going to be bigger than you want. You should tailor an install for your needs. So here we are going to show how to create and use an MSVC Docker container.

The first step is to [install Docker Community Edition](https://docs.docker.com/docker-for-windows/install). If you don’t have Hyper-V enabled, the installer will prompt you to enable it and reboot. After it is installed you need to switch to Windows containers. The official documentation on how to [Install Build Tools into a Container](https://docs.microsoft.com/en-us/visualstudio/install/build-tools-container) walks through this in much more detail. If you refer to that, stop at step 5 and read the following instructions for creating an MSVC Docker container.

Remember that the [VS Build Tools are licensed as a supplement](https://www.visualstudio.com/license-terms/mlt553512/) to your existing Visual Studio license. Any images built with these tools should be for your personal use or for use in your organization in accordance with your existing Visual Studio and Windows licenses. Please don’t share these images on a public Docker hub.

# Visual Studio Build Tools Dockerfiles

To help you get started in creating Dockerfiles tailored for your needs we have [VS Build Tools Dockerfile samples](https://github.com/Microsoft/vs-Dockerfiles) available. Clone that repository locally. There you will find a directory for native-desktop that contains the files we will use as our starting point. If you open that Dockerfile you will see the following line within.

```bash
--add Microsoft.VisualStudio.Workload.VCTools --includeRecommended \
```

That line adds the VCTools workload and additional recommended components that today include CMake and the current Windows SDK. This is a smaller installation than our other sample that includes support for building managed and native projects. To learn more about how you can find the components you need see our [additional workloads and components](http://aka.ms/vs/workloads) documentation.

Open PowerShell in the native-desktop directory or this repo and build the Docker image.

```bash
docker build -t buildtools2017native:latest -m 2GB .
```

The first time you build the image it will pull down the windowsservercore image, download the build tools, and install them into a local container image.

When this is done you can see the images available locally.

```bash
docker images
REPOSITORY                 TAG                            IMAGE ID       CREATED         SIZE
Buildtools2017native       latest                         4ff1cd971254   3 minutes ago   14.9GB
Buildtools2017             latest                         325048ba2240   4 hours ago     22.4GB
microsoft/dotnet-framework 3.5-sdk-windowsservercore-1709 7d89a4baf66c   3 weeks ago     13.2GB
```

While our buildtoolsmsvc image is large, it is significantly smaller than the VS Build Tools image that includes managed support. Note that these image sizes include the base image so they can be deceiving in terms of actual disk space used.

# Trying it all out

Let’s test out our build tools container using a simple little program. Create a new Windows Console Application project in Visual Studio. Add iostream and write out a hello world message as so.

```cpp
#include "stdafx.h"
#include <iostream>
using namespace std;
int main()
{
  cout << "I was built in a container..." << endl;
}
```

We are going to want to deploy this in a future post to a Windows Nano Server container so change the platform target to x64. Go to your project properties and under C/C++ change the Debug Information Format to C7 compatible.

Now we are going to create a build container for our project. Note this line in the Dockerfile we used to create our buildtoolsmsvc image.

```bash
ENTRYPOINT C:\BuildTools\Common7\Tools\VsDevCmd.bat &&
```

The `&&` in that line allows us to pass parameters in when we create a container using this image. Here we will use that to pass in the msbuild command with the options necessary to build our solution. When we start our container for the first time we will map our local source into a directory in the container using the -v option. We will also give the container a name using –name to remember what it is for later. That also makes it easier to run again as we update our source code.

```bash
docker run -v C:\source\ConsoleApplication1:c:\ConsoleApplication1 --name ConsoleApplication1 buildtoolsmsvc msbuild c:\ConsoleApplication1\ConsoleApplication1.sln /p:Configuration=Debug /p:Platform=x64
```

This will spin up a container, use msbuild to build our solution, then exit, and stop the container. You should see output from this process as it runs. Since we mapped to our local volume you will find the build output under your solution directory under x64\Debug. You can run ConsoleApplication1.exe from there and you’ll see the output

```
I was built in a container...
```

As you change your sources you can run the container again by running this command (omit the -a attach option if you don’t need to see the output from within the container).

```bash
docker start -a ConsoleApplication1
```

# Conclusion
This has gotten us a working MSVC toolset in an isolated, easy-to-use Docker container. You can now spin up additional containers for different projects using the approach outlined above. The disk space used by additional containers using the same base image is miniscule compared to spinning up VMs to get isolated build environments.

In a future post we’ll cover how to take the build output from this process, run it in a container, then attach to the process in the container and debug it.

If you are using containers or doing cloud development with C++ we would love to hear from you. We wouldd appreciate it if you could take a few minutes to take our [C++ cloud and container development survey](https://www.surveymonkey.com/r/W35QZCF). This will help us focus on topics that are important to you on the blog and in the form of product improvements.

