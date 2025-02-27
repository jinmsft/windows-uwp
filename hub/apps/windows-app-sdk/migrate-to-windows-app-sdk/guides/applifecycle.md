---
title: Application lifecycle functionality migration
description: This topic contains migration guidance in the application lifecycle area.
ms.topic: article
ms.date: 09/20/2021
keywords: Windows, App, SDK, migrate, migrating, migration, port, porting, application lifecycle, applifecycle, application, lifecycle
ms.author: stwhi
author: stevewhims
ms.localizationpriority: medium
dev_langs:
  - csharp
  - cppwinrt
---

# Application lifecycle functionality migration

This topic contains migration guidance in the application lifecycle area.

## Important APIs

* [**AppInstance**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance) class
* [**Application.OnLaunched**](/windows/winui/api/microsoft.ui.xaml.application.onlaunched) method
* [**AppInstance.GetActivatedEventArgs**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance.getactivatedeventargs) method
* [**ExtendedActivationKind**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.extendedactivationkind) enum

## Summary of API and/or feature differences

Universal Windows Platform (UWP) apps are single-instanced by default; Windows App SDK (WinUI 3) apps are multi-instanced by default.

A UWP app has **App** methods such as **OnFileActivated**, **OnSearchActivated**, and **OnActivated** that implicitly tell you how the app was activated; In a Windows App SDK app, in **App.OnLaunched** (or in any method), call ([**AppInstance.GetActivatedEventArgs**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance.getactivatedeventargs)) to retrieve the activated event args, and check them to determine how the app was activated.

## Single-instanced apps

Universal Windows Platform (UWP) apps are single-instanced by default (you can opt in to support multiple instances&mdash;see [Create a multi-instance UWP app](/windows/uwp/launch-resume/multi-instance-uwp)).

So the way a single-instanced UWP app behaves is that the second (and subsequent) time you launch, your current instance is activated. Let's say for example that in your UWP app you've implemented the file type association feature. If from File Explorer you open a file (of the type for which the app has registered a file type association)&mdash;and your app is already running&mdash;then that already-running instance is activated.

Windows App SDK (WinUI 3) apps, on the other hand, are multi-instanced by default. So, by default, the second (and subsequent) time you launch a Windows App SDK (WinUI 3) app, a new instance of the app is launched. If for example a Windows App SDK (WinUI 3) app implements file type association, and from File Explorer you open a file (of the right type) while that app is already running, then by default a new instance of the app is launched.

If you want your Windows App SDK (WinUI 3) app to be single-instanced like your UWP app is, then you can override the default behavior described above. You can do that in the [**Application.OnLaunched**](/windows/winui/api/microsoft.ui.xaml.application.onlaunched) method of your **App** class.

> [!NOTE]
> Adding the code shown below to **Application.OnLaunched** can simplify your app. However, a lot depends on what else your app is doing. If you're going to end up redirecting, and then terminating the current instance, then you'll want to avoid doing any throwaway work (or even work that needs explicitly undoing). In cases like that, **Application.OnLaunched** might be too late, and you might prefer to do the work in your app's **main** function (or equivalent).

In **OnLaunched**, use [**AppInstance.FindOrRegisterForKey**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance.findorregisterforkey) and [**AppInstance.IsCurrent**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance.iscurrent) to determine whether the current instance is the main instance. If it isn't, then call [**AppInstance.RedirectActivationToAsync**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance.redirectactivationtoasync) to redirect activation to the already-running main instance, and then exit from the current instance (without creating nor activating its main window).

For more info, see [App instancing in AppLifecycle](/windows/apps/windows-app-sdk/applifecycle/applifecycle-instancing).

```csharp
// App.xaml.cs in a Windows App SDK (WinUI 3) app
...
protected override async void OnLaunched(Microsoft.UI.Xaml.LaunchActivatedEventArgs args)
{
    // If this is the first instance launched, then register it as the "main" instance.
    // If this isn't the first instance launched, then "main" will already be registered,
    // so retrieve it.
    var mainInstance = Microsoft.Windows.AppLifecycle.AppInstance.FindOrRegisterForKey("main");

    // If the instance that's executing the OnLaunched handler right now
    // isn't the "main" instance.
    if (!mainInstance.IsCurrent)
    {
        // Redirect the activation (and args) to the "main" instance, and exit.
        var activatedEventArgs =
            Microsoft.Windows.AppLifecycle.AppInstance.GetCurrent().GetActivatedEventArgs();
        await mainInstance.RedirectActivationToAsync(activatedEventArgs);
        System.Diagnostics.Process.GetCurrentProcess().Kill();
        return;
    }

    m_window = new MainWindow();
    m_window.Activate();
}
```

```cppwinrt
// pch.h in a Windows App SDK (WinUI 3) app
...
#include <winrt/Microsoft.Windows.AppLifecycle.h>
...

// App.xaml.h
...
struct App : AppT<App>
{
    ...
    winrt::fire_and_forget OnLaunched(Microsoft::UI::Xaml::LaunchActivatedEventArgs const&);
    ...
}

// App.xaml.cpp
...
using namespace winrt;
using namespace Microsoft::Windows::AppLifecycle;
...
winrt::fire_and_forget App::OnLaunched(LaunchActivatedEventArgs const&)
{
    // If this is the first instance launched, then register it as the "main" instance.
    // If this isn't the first instance launched, then "main" will already be registered,
    // so retrieve it.
    auto mainInstance{ AppInstance::FindOrRegisterForKey(L"main") };

    // If the instance that's executing the OnLaunched handler right now
    // isn't the "main" instance.
    if (!mainInstance.IsCurrent())
    {
        // Redirect the activation (and args) to the "main" instance, and exit.
        auto activatedEventArgs{ AppInstance::GetCurrent().GetActivatedEventArgs() };
        co_await mainInstance.RedirectActivationToAsync(activatedEventArgs);
        ::ExitProcess(0);
        co_return;
    }

    window = make<MainWindow>();
    window.Activate();
}
```

Alternatively, you can call [**AppInstance.GetInstances**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance.getinstances) to retrieve a collection of running [**AppInstance**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance) objects. If the number of elements in that collection is greater than 1, then your main instance is already running, and you should redirect to that.

## File type association

In a Windows App SDK project, to specify the extension point for a file type association, you make the same settings in your `Package.appxmanifest` file as you would for a UWP project. Here are those settings.

Open `Package.appxmanifest`. In **Declarations**, choose **File Type Associations**, and click **Add**. Set the following properties.

**Display name**: MyFile
**Name**: myfile
**File type**: .myf

To register the file type association, build the app, launch it, and close it.

The difference comes in the imperative code. In a UWP app, you implement **App::OnFileActivated** in order to handle file activation. But in a Windows App SDK app, you write code in **App::OnLaunched** to check the extended activation kind ([**ExtendedActivationKind**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.extendedactivationkind)) of the activated event args ([**AppInstance.GetActivatedEventArgs**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance.getactivatedeventargs)), and see whether the activation is a file activation.

> [!NOTE]
> Don't use the [**Microsoft.UI.Xaml.LaunchActivatedEventArgs**](/windows/winui/api/microsoft.ui.xaml.launchactivatedeventargs) object passed to **App::OnLaunched** to determine the activation kind, because it reports "Launch" unconditionally.

If your app has navigation, then you'll already have navigation code in **App::OnLaunched**, and you might want to re-use that logic. For more info, see [Do I need to implement page navigation?](winui3.md#do-i-need-to-implement-page-navigation).

```csharp
// App.xaml.cs in a Windows App SDK app
...
using Microsoft.Windows.AppLifecycle;
...
protected override void OnLaunched(Microsoft.UI.Xaml.LaunchActivatedEventArgs args)
{
    var activatedEventArgs = Microsoft.Windows.AppLifecycle.AppInstance.GetCurrent().GetActivatedEventArgs();
    if (activatedEventArgs.Kind == Microsoft.Windows.AppLifecycle.ExtendedActivationKind.File)
    {
        ...
    }
    ...
}
```

```cppwinrt
// pch.h in a Windows App SDK app
...
#include <winrt/Microsoft.Windows.AppLifecycle.h>

// App.xaml.cpp
...
using namespace Microsoft::Windows::AppLifecycle;
...
void App::OnLaunched(LaunchActivatedEventArgs const&)
{
    auto activatedEventArgs{ AppInstance::GetCurrent().GetActivatedEventArgs() };
    if (activatedEventArgs.Kind() == ExtendedActivationKind::File)
    {
        ...
    }
    ...
}
```

## OnActivated, OnBackgroundActivated, and other activation-handling methods

In a UWP app, to override the various means by which your app can be activated, you can override corresponding methods on your **App** class, such as **OnFileActivated**, **OnSearchActivated**, or the more general **OnActivated**.

In a Windows App SDK app, in **App.OnLaunched** (or in fact at any time) you can call ([**AppInstance.GetActivatedEventArgs**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.appinstance.getactivatedeventargs)) to retrieve the activated event args, and check them to determine how the app was activated.

See the [File type association](#file-type-association) section above for more details and a code example. You can apply the same technique for any activation kind specified by the [**ExtendedActivationKind**](/windows/windows-app-sdk/api/winrt/microsoft.windows.applifecycle.extendedactivationkind) enum.

## Related topics

* [App instancing in AppLifecycle](/windows/apps/windows-app-sdk/applifecycle/applifecycle-instancing)
* [Do I need to implement page navigation?](winui3.md#do-i-need-to-implement-page-navigation)
