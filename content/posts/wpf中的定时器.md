---
title: WPF 中的定时器
slug: "timers-in-wpf"
date: 2026-11-10T17:36:28+08:00
categories: 技术
tags: 
  - wpf
  - 笔记
draft: false
description: "记录了 WPF 中 DispatcherTimer、System.Timers.Timer 与 PeriodicTimer 三种定时器的运行上下文、UI 更新方式、回调重叠风险及适用场景。"
---

在此列举了一些，我个人觉得常用的定时器，不包含所有的定时器。

| 定时器                | 运行上下文            | 直接更新 UI | 回调重叠风险     | 适用场景                     |
| :-------------------- | :-------------------- | :---------- | :--------------- | :--------------------------- |
| `DispatcherTimer`     | UI 线程 (Dispatcher)  | 可以        | 无（但会阻塞 UI） | 轻量级 UI 刷新（时钟、状态） |
| `System.Timers.Timer` | 后台线程 (ThreadPool) | 不可以      | 有               | 后台复杂逻辑、耗时操作       |
| `PeriodicTimer`       | 异步等待 (await)      | 不可以      | 无（顺序处理）   | 异步后台周期任务             |

## DispatcherTimer

[DispatcherTimer](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.threading.dispatchertimer) 是专为 WPF 设计的，它的 `Tick` 事件直接在 WPF 的 UI 线程（Dispatcher） 上执行。

这意味了在事件执行中，可以直接操作任何 UI 控件，而无需使用 Dispatcher.Invoke。因此非常适合一些轻量级的 UI 更新，比如时钟显示、进度条刷新或状态栏文字切换等。

```csharp
using System.Windows.Threading;

// 简单示例：每秒更新一次时间显示
var timer = new DispatcherTimer
{
    Interval = TimeSpan.FromSeconds(1)
};
timer.Tick += (s, e) 
    => myTextBlock.Text = DateTime.Now.ToString("HH:mm:ss");
timer.Start();
```

> [!NOTE]
>
> 如果 `Tick` 事件中的处理逻辑耗时过长，会直接阻塞 UI 线程，导致界面卡顿。因此，它只适合轻量操作。

## System.Timers.Timer

[System.Timers.Timer](https://learn.microsoft.com/zh-cn/dotnet/api/system.timers.timer) 的 `Elapsed` 事件在后台线程上触发，这使得它摆脱了 UI 线程的束缚，适合执行一些较复杂的逻辑、耗时的操作，不会因此阻塞界面。

而代价是线程安全问题，在 `Elapsed` 中不能直接修改 UI 控件，必须通过 `Dispatcher.Invoke` 或 `Dispatcher.InvokeAsync` 将 UI 更新操作跳转到 UI 线程上执行。

```csharp
using Timer = System.Timers.Timer;

var timer = new Timer(1000);
timer.Elapsed += (s, e) =>
{
    var data = DoHeavyWork(); // 后台耗时计算
    Application.Current.Dispatcher.Invoke(() =>
    {
        myTextBlock.Text = data; // UI 线程更新
    });
};
timer.Start();
```

> [!NOTE]
>
> 如果 `Elapsed` 的处理时间超过了 `Interval`，下一次回调可能会在前一次尚未完成时就开始，导致回调重叠。

## PeriodicTimer

[PeriodicTimer](https://learn.microsoft.com/zh-cn/dotnet/api/system.threading.periodictimer) 是从 .NET 6 开始引入的异步友好型定时器，不再使用回调，而是通过 `await WaitForNextTickAsync()` 来等待下一个 `Tick`。

这种模式天然的避免了回调重叠的问题，因为下一次是等待上一次处理完成之后才开始，同时可以用 `CancellationToken` 优雅地取消循环。

```csharp
using System.Threading;

async Task RunPeriodicAsync(CancellationToken ct)
{
    using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
    while (await timer.WaitForNextTickAsync(ct))
    {
        var data = await FetchDataAsync(ct); // 异步后台操作
        await Application.Current.Dispatcher.InvokeAsync(() =>
        {
            myTextBlock.Text = data; // 更新 UI
        });
    }
}
```