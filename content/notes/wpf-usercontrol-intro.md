---
title: 自定义控件：初识 UserControl
date: 2026-05-16T20:55:46+08:00
categories: 
  - 技术
  - 学习
tags:
  - wpf
  - csharp
  - 笔记
draft: false
description: "简述了，自己对 UserControl 的理解，以及如何创建。"
---


> 参考链接：[控件编写概述 - WPF | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/controls/control-authoring-overview#models-for-control-authoring)

## 用户控件是什么

经过对**自定义控件**的学习后，我对“[用户控件](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.usercontrol)”（`UserControl`）方式，有了大致的理解。

- 用户控件将多个现有控件组合成一个整体，作为一个复合控件使用。
- 而这个特性，让我意识到，可以将一个复杂的界面，拆分成一个个独立小组件。
- 也就是说通过定义界面布局和逻辑，实现 UI 组件的模块化设计和代码复用。

```text
界面/
  |- 用户控件
    |- 现有控件
    |- 现有控件
    |- ...
  |- 用户控件
  |- ...
```

> 可能我理解的不是恰当，如果有问题的话， 希望能得到指正。

## 创建用户控件

然后创建这个用户控件也很容易。打开 JetBrains Rider，在 WPF 项目中：

```text
右键项目 -> 添加 -> 用户控件
```

就会生成一个 XAML 文件（`.xaml`）和一个对应的 C# 后台代码文件（`.xaml.cs`）。

与平常的创建的窗口文件区别不大，看起来就是将 `<Window></Window>` 换成了 `<UserControl></UserControl>`。依旧是在 XAML 文件中定义控件的外观，后台代码中定义控件的行为和逻辑。

{{< tabs >}}

{{< tab name="XAML" >}}

```xml {hl_lines=[1]}
<UserControl x:Class="MyApp.MyControl"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             mc:Ignorable="d"
             d:DesignHeight="300" d:DesignWidth="300">
    <Grid>
        
    </Grid>
</UserControl>
```

{{< /tab >}}

{{< tab name="C#" >}}

```csharp {hl_lines=[6]}
using System.Windows;
using System.Windows.Controls;

namespace MyApp;

public partial class MyControl : UserControl
{
    public MyControl()
    {
        InitializeComponent();
    }
}
```

{{< /tab >}}

{{< /tabs >}}