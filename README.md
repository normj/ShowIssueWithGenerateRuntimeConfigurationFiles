# Purpose

This repository is to demonstrate a potential bug in the .NET Core 3.1 SDK that breaks the AWS Lambda .NET tooling.

## Background

The AWS .NET Lambda tooling explicitly sets the **GenerateRuntimeConfigurationFiles** to true flag when doing a `dotnet publish`. This was necessary because in Lambda we send the entry point is a netcoreapp2.1 Library but we need to make sure the runtime.config is generated with the deployment bundle even though it is library so on Lambda's side we have the correct start up parameters. In older versions of the .NET SDK it would not generate the runtime.config file if the project doing `dotnet publish` on was a library which is why we explicitly set the GenerateRuntimeConfigurationFiles to true.

## The breaking issue

I have found with .NET Core 3.1 SDK installed if I have a netcoreapp2.1 project that references a netstandard2.0 project then the `dotnet publish /p:GenerateRuntimeConfigurationFiles=true` command fails with the following error.

```
> dotnet publish /p:GenerateRuntimeConfigurationFiles=true
Microsoft (R) Build Engine version 16.4.0-preview-19529-02+0c507a29b for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 46.82 ms for C:\temp\testruntimeconfig\net31sdk\NetCoreClassLib\NetCoreClassLib.csproj.
  Restore completed in 46.82 ms for C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj.
  You are using a preview version of .NET Core. See: https://aka.ms/dotnet-core-preview
C:\Program Files\dotnet\sdk\3.1.100-preview3-014645\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.Sdk.targets(255,5): error MSB4018: The "GenerateRuntimeConfigurationFiles" task failed unexpectedly. [C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj]
C:\Program Files\dotnet\sdk\3.1.100-preview3-014645\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.Sdk.targets(255,5): error MSB4018: System.NullReferenceException: Object reference not set to an instance of an object. [C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj]
C:\Program Files\dotnet\sdk\3.1.100-preview3-014645\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.Sdk.targets(255,5): error MSB4018:    at Microsoft.NET.Build.Tasks.GenerateRuntimeConfigurationFiles.AddFrameworks(RuntimeOptions runtimeOptions, ProjectContext projectContext) [C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj]
C:\Program Files\dotnet\sdk\3.1.100-preview3-014645\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.Sdk.targets(255,5): error MSB4018:    at Microsoft.NET.Build.Tasks.GenerateRuntimeConfigurationFiles.WriteRuntimeConfig(ProjectContext projectContext) [C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj]
C:\Program Files\dotnet\sdk\3.1.100-preview3-014645\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.Sdk.targets(255,5): error MSB4018:    at Microsoft.NET.Build.Tasks.GenerateRuntimeConfigurationFiles.ExecuteCore() [C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj]
C:\Program Files\dotnet\sdk\3.1.100-preview3-014645\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.Sdk.targets(255,5): error MSB4018:    at Microsoft.NET.Build.Tasks.TaskBase.Execute() [C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj]
C:\Program Files\dotnet\sdk\3.1.100-preview3-014645\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.Sdk.targets(255,5): error MSB4018:    at Microsoft.Build.BackEnd.TaskExecutionHost.Microsoft.Build.BackEnd.ITaskExecutionHost.Execute() [C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj]
C:\Program Files\dotnet\sdk\3.1.100-preview3-014645\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.Sdk.targets(255,5): error MSB4018:    at Microsoft.Build.BackEnd.TaskBuilder.ExecuteInstantiatedTask(ITaskExecutionHost taskExecutionHost, TaskLoggingContext taskLoggingContext, TaskHost taskHost, ItemBucket bucket, TaskExecutionMode howToExecuteTask) [C:\temp\testruntimeconfig\net31sdk\NetStandardLib\NetStandardLib.csproj]
```

If I add a globaljson file and force the .NET Core 3.0 SDK the command works fine.

## How to reproduce

On a machine that has both .NET Core 3.0 and .NET Core 3.1 SDK's installed. 

* Clone this repository
* cd to ./net30sdk/NetCoreClassLib
** Notice now NetCoreClassLib has a project dependency to a .NET Standard 2.0 project
* Execute the command `dotnet publish /p:GenerateRuntimeConfigurationFiles=true`
* The command works correctly because there is a global.json forcing .NET Core 3.0 SDK
* cd ../../net30sdk/NetCoreClassLib
* Execute the command `dotnet publish /p:GenerateRuntimeConfigurationFiles=true`
* This time the dotnet publish fails with the error message "error MSB4018: The "GenerateRuntimeConfigurationFiles" task failed unexpectedly."

## Solutions

To ensure .NET developers who install .NET Core 3.1 either on purpose or due to updating their Visual Studio when .NET Core 3.1 goes GA we need the `dotnet publish` to not have a change in behavior with GenerateRuntimeConfigurationFiles. We could potentially change the AWS .NET Lambda tooling to no longer set the GenerateRuntimeConfigurationFiles flag when publishing as it seems current versions of the .NET Core SDK always right the runtime.config file but that won't help users that don't know to update.
