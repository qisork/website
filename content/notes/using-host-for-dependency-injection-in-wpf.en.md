---
title: Using Host for Dependency Injection in WPF
date: 2026-04-06T11:21:35+08:00
tags:
  - Note
  - CSharp
  - Wpf
draft: false
description: This article records how to implement dependency injection in WPF applications using Microsoft.Extensions.Hosting, including configuration steps, three service lifetime management methods, and practical application examples.
summary: This article records how to implement dependency injection in WPF applications using Microsoft.Extensions.Hosting, including configuration steps, three service lifetime management methods, and practical application examples.
---

Runtime Environment:

- .NET 10
- Microsoft.Extensions.Hosting 10.0.5

## What is Dependency Injection

Dependency injection, as the name suggests, is a design pattern that delegates the management of object dependencies to external sources and injects them when needed, replacing internal instantiation to reduce coupling.

In .NET, [Host](https://www.nuget.org/packages/Microsoft.Extensions.Hosting) is used to manage object dependencies, along with features such as configuration management and logging. This article primarily focuses on dependency injection, without going into excessive detail about other features.

## How to Use Dependency Injection in WPF

In WPF, the application entry point is `App.xaml.cs`. To allow Host to span the entire lifecycle of a WPF application, we need to build and start the Host here.

### Prerequisites

Install the necessary NuGet package, as .NET does not include Host by default and requires manual addition.

```shell
dotnet add package Microsoft.Extensions.Hosting --version 10.0.5
```

{{% steps %}}

### Modify `App.xaml`

Remove the **default startup window** and let Host manage it.

```xml {hl_lines=4}
<Application x:Class="WpfApp1.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <!-- Remove the default StartupUri="MainWindow.xaml" -->
</Application>
```

### Modify `App.xaml.cs`

In the code-behind, register objects that require dependency injection into the Host, then build and start the Host.

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System.Windows;

namespace WpfApp1;

public partial class App : Application
{
    // Global host: the only core object for the entire application
    public static IHost AppHost { get; private set; }

    // Application startup event
    protected override async void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // 1. Create Host builder
        var builder = Host.CreateApplicationBuilder();

        // 2. Register all windows and services
        ConfigureServices(builder.Services);

        // 3. Build the host
        AppHost = builder.Build();

        // 4. Start the host (non-blocking mode)
        await AppHost.StartAsync();

        // 5. Get registered objects from the Host
        var mw = AppHost.Services.GetRequiredService<MainWindow>();

        // 6. Display the main window
        mw.Show();
    }

    // Service registration: write all objects that need automatic creation here
    private void ConfigureServices(IServiceCollection services)
    {
        // Register main window
        services.AddSingleton<MainWindow>();
    }

    // Application exit event
    protected override async void OnExit(ExitEventArgs e)
    {
        await AppHost.StopAsync();
        AppHost.Dispose();
        
        base.OnExit(e);
    }
}
```

{{% /steps %}}

## Three Service Lifetimes Supported by Host

Regarding object dependencies, Host supports three service lifetimes:

- **Transient**: Creates a new instance for each request
- **Scoped**: Shares one instance within each scope
- **Singleton**: Shares one instance throughout the entire application lifetime

In the code above, the singleton lifetime is used, meaning only one instance will be created during the entire WPF application lifecycle.

```csharp
// 1. Transient: creates a new object each time it's used
services.AddTransient<MyTool>();

// 2. Scoped: shares one object within a scope
services.AddScoped<MainViewModel>();

// 3. Singleton: creates only 1 object for the entire application
services.AddSingleton<MainWindow>();
```

When registering object dependencies, you can specify an interface and its concrete implementation class. Taking transient lifetime as an example:

```csharp
services.AddTransient<IMyTool, MyTool>();
```

When detecting a need to inject a dependency of type `IMyTool`, it will instantiate an object of type `MyTool` and provide it to the requester.

## Simple Usage Demonstration

Inject a `ViewModel` into `MainWindow` as its data context.

{{% steps %}}

### Create a New `ViewModel`

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace WpfApp1;

public class MainWindowViewModel : INotifyPropertyChanged
{
    // ... specific content omitted ...
}
```

### Register in `ConfigureServices()`

```csharp
services.AddSingleton<MainWindow>();
services.AddTransient<MainWindowViewModel>();
```

### Constructor Injection in `MainWindow`

```csharp {hl_lines=7}
using System.Windows;

namespace WpfApp1;

public partial class MainWindow : Window
{
    public MainWindow(MainWindowViewModel viewModel)
    {
        InitializeComponent();
        DataContext = viewModel;
    }
}
```

{{% /steps %}}