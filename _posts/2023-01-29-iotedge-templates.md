---
layout: post
title: Custom templates for Azure IoT Edge modules
comments: true
---

I've been working on a few Azure IoT Edge projects in the last 2 years.  During that period, I always felt that the IoT Edge module projects that are scaffolded by the standard project template are very simplistic.

The default project template for IoT Edge projects in Visual Studio.NET generates a console program that writes logs via `Console.WriteLine` for instance.
However, I don't want that. I want to use the `ILogger` functionality instead so that I'm able to conditionally log traces based on their log-level.  
Next to that, there are some common tasks that you typically want to perform in an IoT Edge module, such as reading information from the module-twin.  I feel that such common tasks can be generated for you so that you do not need to write the same code over and over again for every IoT Edge module that you develop.
Therefore, I've decided to create my own IoT Edge template.

# Some background

Initially, I wanted to integrate my IoT Edge template in Visual Studio.NET to have a seamless experience but unfortunately that is not possible.  For more details on this see [this](https://github.com/microsoft/vs-azure-iot-edge-docs/issues/42) issue on GitHub.
Since I've learned [here](https://github.com/microsoft/vs-azure-iot-edge-docs/issues/42#issuecomment-1352222428) that the recommended way to create a new (IoT Edge) project is via the CLI (`dotnet new`).

# Features

When I create an IoT Edge module, I typically want to make use of these features:

- Easily read desired properties from the Module Twin and preferably abstract this away via a `Configuration` class;

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
- You'll be asked the question if you want to run the `AddModuleToDeploymentManifest.ps1` script.  This script will search for the `deployment.template.json` file in your solution and make the  changes to that file that are required to include the new IoT Edge module in the deployment manifest.

This project uses a hosted environment for the IoT Edge module that allows you to easily use background services.  If you want a more simple approach for your IoT Edge module, specify the `--no-backgroundservices` parameter.

Normally, you will create a new IoT Edge module project and add it to an IoT Edge solution.  The IoT Edge solution consists of a project that contains the deployment manifest.
When you execute the `fgiotedgemodule` template from the location where the `.sln` file of the IoT Edge solution is located, the template will also make the necessary changes to the `deployment.template.json` file.

## Parameters

These parameters are available when creating an IoT Edge module project using the `fgiotedgemodule` template:

|parameter|description
|-|-|
|repository|The address of the image container repository.  If not specified, the `module.json` file specifies the repository as a variable that can be set / replaced during deployment.
|no-backgroundservices|Specifies that a module must be generated which does not make use of hosted background-services.  (Default: false)

## What do you get ?

After creating a new project using the `fgiotedgemodule` template, you'll end up with a C# project that has scaffolded a skeleton for your new IoT Edge module.

When we quickly go over the generated project, we'll see that a `HostBuilder` is used to create an `IHost` implementation that is responsible for running the application.
Taking a closer look to the `HostBuilder` reveals that:

- A `ModuleClient` instance is registered to the Dependency Injection container via the `ConfigureIoTEdgeModuleClient` extension method that is defined in the `Fg.IoTEdgeModule` nuget package.  This extension method sets up a `ModuleClient` instance using the specified protocol and registers this `ModuleClient` instance as a singleton in the DI container.  The `ModuleClient` is also configured to make sure that the module will restart when the desired properties in the Module Twin have changed.  Of course, it is possible to implement other logic for this callback.

- A Configuration object is initialized with the desired properties that can be found in the Module Twin.  You'll find a Configuration-class that is called `<MyModuleName>Configuration` for this.  Be sure to add some custom code in the `InitializeFromTwin` and `SetReportedProperties` methods in that class for reading properties from the Module Twin and reporting changes back to the Module Twin.

- Logging is configured to make use of the `ILogger` infrastructure.

As mentionned above, the project also contains a `<MyModuleName>Configuration` class.  This class can be used to abstract away reading the desired properties from the Module Twin and setting the reported properties of the Module Twin.  To do this, add additional code to the `InitializeFromTwin` method for reading desired properties from the Module Twin.  Additionally, add code to the `SetReportedProperties` to make sure that settings can be reported back to the reported Properties section of the Module Twin.
The configuration class is also registered as a singleton in the DI container, so it can be injected in other classes when needed.
Use the `UpdateReportedPropertiesAsync` method of the configuration object to report the configuration properties to the ModuleTwin whenever you want.

An example of a BackgroundService is also generated.  This background service can be used to implement the actual work that must be done by the IoT Edge module.

You'll also see that a `ShutdownHandler` is created.  This class contains a `CancellationTokenSource` that can be used to stop the IoT Edge module.  The `ShutdownHandler` will make sure that the IoT Edge module gracefully stops executing.

Finally, you'll also find some `Dockerfiles` that will be used to build a container image for your IoT Edge module.

## Conclusion

I firmly believe that this custom template for IoT Edge modules offers some advantages over the out-of-the-box template that comes with Visual Studio.
The goal is that developers who create IoT Edge solutions can benefit from some best practices that this template offers so that they can focus on writing their actual business logic.

I really hope that you benefit from this template as well.  Should you find any issues in it or have questions about it, don't hesitate to open an issue [in the Github repository of this project](https://github.com/fgheysels/Fg.IoTEdge.Template).