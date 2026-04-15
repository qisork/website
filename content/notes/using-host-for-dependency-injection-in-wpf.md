---
title: 通过 Host 在 WPF 中进行依赖注入
date: 2026-04-06T11:21:35+08:00
tags:
  - Note
  - CSharp
  - Wpf
draft: false
description: 本文记录了如何在 WPF 应用中使用 Microsoft.Extensions.Hosting 实现依赖注入，包括配置步骤、三种服务生命周期管理及实际应用示例。
summary: 本文记录了如何在 WPF 应用中使用 Microsoft.Extensions.Hosting 实现依赖注入，包括配置步骤、三种服务生命周期管理及实际应用示例。
---

运行环境：

- .NET 10
- Microsoft.Extensions.Hosting 10.0.5

## 什么是依赖注入

依赖注入，顾名思义是将对象的依赖交由外部管理，然后在需要的时候添加进来，替代内部实例化，以降低耦合的设计模式。

在 .NET 中使用 [Host](https://www.nuget.org/packages/Microsoft.Extensions.Hosting) 来管理对象依赖，同时还包括配置管理和日志记录等功能，在这里主要介绍依赖注入，其他的不过多赘述。

## 如何在 WPF 中使用依赖注入

在 WPF 中，程序的启动入口是 `App.xaml.cs`，要想让 Host 贯穿整个 WPF 软件的生命周期，我们需要在这里构建 Host，然后启动。

### 前置条件

安装必要的 NuGet 包，这是因为 .NET 不自带 Host，需要手动添加。

```shell
dotnet add package Microsoft.Extensions.Hosting --version 10.0.5
```

{{% steps %}}

### 修改 `App.xaml`

删除**默认启动窗口**，交给 Host 管理。

```xml {hl_lines=4}
<Application x:Class="WpfApp1.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <!-- 删掉默认的 StartupUri="MainWindow.xaml" -->
</Application>
```

### 修改 `App.xaml.cs`

在后置代码中，将需要依赖注入的对象，注册到 Host 中，然后构建并启动 Host。

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using System.Windows;

namespace WpfApp1;

public partial class App : Application
{
    // 全局 Host：整个应用唯一的核心对象
    public static IHost AppHost { get; private set; }

    // 应用启动事件
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        // 1. 创建 Host 程序构建器
        var builder = Host.CreateApplicationBuilder();

        // 2. 注册所有窗口、服务
        ConfigureServices(builder.Services);

        // 3. 构建 Host
        AppHost = builder.Build();

        // 4. 启动 Host
        AppHost.Start();

        // 5. 从 Host 中获取已注册的对象
        var mw = AppHost.Services.GetRequiredService<MainWindow>();

        // 6. 显示主窗口
        mw.Show();
    }

    // 服务注册：所有需要自动创建的对象写这里
    private void ConfigureServices(IServiceCollection services)
    {
        // 注册主窗口
        services.AddSingleton<MainWindow>();
    }

    // 应用关闭事件
    protected override void OnExit(ExitEventArgs e)
    {
        // 关闭 Host，并释放资源
        AppHost.StopAsync().Wait();;
        AppHost.Dispose();
        
        base.OnExit(e);
    }
}
```

{{% /steps %}}

## Host 支持三种服务生命周期

关于对象依赖，Host 支持三种服务生命周期：

- **瞬时 (Transient)**：每次请求创建新实例
- **作用域 (Scoped)**：每个作用域内共享一个实例
- **单例 (Singleton)**：整个应用生命周期内共享一个实例

在上面的代码中，正是使用了单例生命周期，在整个 WPF 应用生命周期里，只会创建一个实例。

```csharp
// 1. 瞬时：每次用都创建新对象
services.AddTransient<MyTool>();

// 2. 作用域：一个作用域内共用一个
services.AddScoped<MainViewModel>();

// 3. 单例：整个应用只创建 1 个对象
services.AddSingleton<MainWindow>();
```

注册对象依赖时，可以指定接口和具体的实现类，以添加瞬时生命周期为例：

```csharp
services.AddTransient<IMyTool, MyTool>();
```

当检测到需要注入 `IMyTool` 类型的依赖时，就会实例化 `MyTool` 类型的对象，给需要者。

## 简单的使用演示

给 `MainWindow` 注入 `ViewModel`，作为数据上下文。

{{% steps %}}

### 新建 `ViewModel`

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

namespace WpfApp1;

public class MainWindowViewModel : INotifyPropertyChanged
{
    // ... 具体内容省略 ...
}
```

### 在 `ConfigureServices()` 注册

```csharp
services.AddSingleton<MainWindow>();
services.AddTransient<MainWindowViewModel>();
```

### `MainWindow` 构造函数注入

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