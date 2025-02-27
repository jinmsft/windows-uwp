---
description: Learn about using the bootstrapper API to reference the Windows App SDK framework package at run time.
title: Reference the Windows App SDK framework package at run time
ms.topic: article
ms.date: 07/30/2021
ms.localizationpriority: medium
---

# Reference the Windows App SDK framework package at run time

Unpackaged apps (that is, apps that do not use MSIX for their deployment technology) must use the *bootstrapper API* before they can use Windows App SDK features such as WinUI, App lifecycle, MRT Core, and DWriteCore. The bootstrapper API enables unpackaged apps to dynamically take a dependency on the Windows App SDK *framework package* at run time.

For background information about framework packages, see [MSIX framework packages and dynamic dependencies](../desktop/modernize/framework-packages/framework-packages-overview.md).

> [!NOTE]
> In addition to the bootstrapper API, the Windows App SDK also provides an implementation of the *dynamic dependency API*. This API enables your unpackaged apps to take a dependency on *any* framework package (not just the Windows App SDK framework package), and it is used internally by the bootstrapper API. For more information about the dynamic dependency API, see [Reference framework packages at run time](../desktop/modernize/framework-packages/use-the-dynamic-dependency-api.md).

## Using the bootstrapper API

The bootstrapper API consists of two C/C++ functions that are declared in the [mddbootstrap.h](/windows/windows-app-sdk/api/win32/mddbootstrap) header file in the Windows App SDK: [MddBootstrapInitialize](/windows/windows-app-sdk/api/win32/mddbootstrap/nf-mddbootstrap-mddbootstrapinitialize) and [MddBootstrapShutdown](/windows/windows-app-sdk/api/win32/mddbootstrap/nf-mddbootstrap-mddbootstrapshutdown). These functions are provided by the [bootstrapper](deployment-architecture.md#bootstrapper) library in the Windows App SDK. This library is a small DLL that must be distributed with your app; it is not part of the framework package itself.

For a code example that demonstrates how to use the bootstrapper API to initialize the Windows App SDK runtime, see [Build and deploy an unpackaged app that uses the Windows App SDK](tutorial-unpackaged-deployment.md). For additional details, see the [dynamic dependencies specification](https://github.com/microsoft/WindowsAppSDK/blob/main/specs/dynamicdependencies/DynamicDependencies.md) on GitHub.

### MddBootstrapInitialize

This function initializes the calling process to use the version of the Windows App SDK framework package that best matches the criteria that you pass to the function parameters. Typically, this results in referencing the version of the framework package that matches the Windows App SDK NuGet package that is installed. If multiple packages meet the criteria, the best candidate is selected. This function must be one of the first calls in the app's startup to ensure the bootstrapper component can properly initialize the Windows App SDK and add the run-time reference to the framework package.

This function also initializes the [Dynamic Dependency Lifetime Manager (DDLM)](deployment-architecture.md#dynamic-dependency-lifetime-manager-ddlm). This component provides infrastructure to prevent the OS from servicing the Windows App SDK framework package while it is being used by an unpackaged app.

### MddBootstrapShutdown

The function removes changes to the current process made by [MddBootstrapInitialize](/windows/windows-app-sdk/api/win32/mddbootstrap/nf-mddbootstrap-mddbootstrapinitialize). After this function is called, your app can no longer call Windows App SDK APIs, including the dynamic dependencies API.

This function also shuts down the [Dynamic Dependency Lifetime Manager (DDLM)](deployment-architecture.md#dynamic-dependency-lifetime-manager-ddlm) so that Windows can service the framework package as necessary.

## .NET wrapper for the bootstrapper API

Although you can call the C/C++ bootstrapper API directly from .NET apps, this requires the use of [platform invoke](/dotnet/framework/interop/consuming-unmanaged-dll-functions) to call the functions. For an example that demonstrates how to do this, see the C# instructions for 1.0 Preview 1 and earlier releases in [Build and deploy an unpackaged app that uses the Windows App SDK](tutorial-unpackaged-deployment.md?tabs=csharp-dotnet-preview1#instructions).

In Windows App SDK 1.0 Preview 2 and later releases, you can simplify this process by using the .NET wrapper for the bootstrapper API available in the Microsoft.WindowsAppRuntime.Bootstrap.Net.dll assembly. This assembly provides an easier and more natural API for .NET developers to access the bootstrapper's functionality. The `Bootstrap` class provides static `Initialize`, `TryInitialize`, and `Shutdown` functions that wrap calls to the unmanaged [MddBootstrapInitialize](/windows/windows-app-sdk/api/win32/mddbootstrap/nf-mddbootstrap-mddbootstrapinitialize) and [MddBootstrapShutdown](/windows/windows-app-sdk/api/win32/mddbootstrap/nf-mddbootstrap-mddbootstrapshutdown) functions for most common scenarios. For an example that demonstrates how to use the .NET wrapper for the bootstrapper API, see the C# instructions for 1.0 Preview 2 and later releases in [Build and deploy an unpackaged app that uses the Windows App SDK](tutorial-unpackaged-deployment.md?tabs=csharp-dotnet-preview2#instructions).

For more information about the .NET wrapper for the bootstrapper API, see these resources:

- [Section 6.1.4](https://github.com/microsoft/WindowsAppSDK/blob/main/specs/dynamicdependencies/DynamicDependencies.md#614-microsoftwindowsapplicationmodeldynamicdependency-c) of the dynamic dependencies specification.
- [Bootstrap.cs](https://github.com/microsoft/WindowsAppSDK/blob/main/dev/Bootstrap/CS/Microsoft.WindowsAppRuntime.Bootstrap.Net/Bootstrap.cs): The open source implementation of the .NET wrapper for the bootstrapper API.

## Related topics

- [Windows App SDK deployment guide for unpackaged apps](deploy-unpackaged-apps.md)
- [Dynamic dependencies specification](https://github.com/microsoft/WindowsAppSDK/blob/main/specs/dynamicdependencies/DynamicDependencies.md)
- [Runtime architecture for the Windows App SDK](deployment-architecture.md)
- [Build and deploy an unpackaged app that uses the Windows App SDK](tutorial-unpackaged-deployment.md)
