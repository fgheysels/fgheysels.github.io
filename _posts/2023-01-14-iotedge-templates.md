---
layout: post
title: Custom templates for Azure IoT Edge modules
comments: true
---

I've been working on a few Azure IoT Edge projects in the last 2 years.  During that period, I always felt that the IoT Edge module projects that are scaffolded by the standard project template are very simplistic.
Instead of logging output via `Console.Write`, I want to use the `ILogger` functionality. I also wanted an easy way to read information from the module-twin, etc...
Therefore, I've decided to create my own IoT Edge template.

# Some background

Initially, I wanted to integrate my IoT Edge template in Visual Studio.NET to have a seamless experience but unfortunately that is not possible.  For more details on this see [this](https://github.com/microsoft/vs-azure-iot-edge-docs/issues/42) issue on GitHub.
Since I've learned [here](https://github.com/microsoft/vs-azure-iot-edge-docs/issues/42#issuecomment-1352222428) that the recommended way to create a new (IoT Edge) project is via the CLI (`dotnet new`).

# Features

When I create an IoT Edge module, I typically want to make use of these features:

- Easily read desired properties from the Module Twin and preferably abstract this away via a 'Configuration' class;

- Log traces via the `ILogger` infrastructure and easily modify the configured loglevel (via the desired properties in the Module Twin)

- Have the ability to make use of background-services and dependency injection;

I've already written functionality for some of these and bundled them in the `Fg.IoTEdgeModule` [project](https://github.com/fgheysels/Fg.IoTEdgeModule).  However, I got a bit tired of writing the same plumbing code from scratch so therefore I've created a custom IoT Edge project template that uses this library.

# How to use it

At this moment, there exists no published template package for this template, so if you want to use it, you need to get the source code of the template and install the template to your machine from there:

- `git clone https://github.com/fgheysels/Fg.IoTEdge.Template.git`
- from the `src/Fg.IoTEdge.Module` directory, execute `dotnet new install ./`

Now the template should be available on your system.  When you execute `dotnet new list` you should see an entry with a shortname `fgiotedgemodule`.

- Navigate to the folder where you want to create a new IoT Edge project
- Execute `dotnet new fgiotedgemodule --name <myModuleName>`.  A new IoT Edge project will be scaffolded in this location.  
This project uses a hosted environment for the IoT Edge module that allows you to easily use background services.  If you want a more simple approach for your IoT Edge module, specify the `--no-backgroundservices` parameter.

Normally, you will create a new IoT Edge module project and add it to an IoT Edge solution.  The IoT Edge solution consists of a project that contains the deployment manifest.
When you execute the `fgiotedgemodule` template from the location where the `.sln` file of the IoT Edge solution is located, the template will also make the necessary changes to the `deployment.template.json` file.

## Parameters

These parameters are available when creating an IoT Edge module project using the `fgiotedgemodule` template:

|parameter|description
|-|-|
|repository|The address of the image container repository.  If not specified, the `module.json` file specifies the repository as a variable that can be set / replaced during deployment.
|no-backgroundservices|Specifies that a module must be generated which does not make use of hosted background-services.  (Default: false)

