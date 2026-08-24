---
title: WPF 资源字典
slug: wpf-resource-dictionary
aliases:
  - /notes/wpf-resource-dictionary/
date: 2026-07-29T15:03:40+08:00
categories: 
  - 技术
tags:
  - wpf
  - csharp
  - 笔记
draft: false
description: "本文记录了我对于WPF的资源字典的一些了理解，以及资源字典的使用。"
---

对我来说，资源字典类似一个单独的 CSS 文件，里面存放了与实际代码分离的样式代码，用来定义组件的外观，以及额外的动效，然后根据需要单独引用，即可应用定义好的样式。

使用资源字典的好处就是，与实际代码分离后，可以集中管理，同时提高复用性，比如多个控件公用同一套样式。

最后，我对于”资源字典“这个名字，还有一个疑惑，资源我能直接理解，样式也是一种资源，类似图片、音乐和字体等等，但为什么还要加上一个”字典“二字呢？

后续，经过我的查阅，算是明白了为什么。

{{< callout type="info" >}}

在 WPF 中，`ResourceDictionary` 之所以被称为“资源字典”，是因为它在**数据结构**、**设计模式**和**功能语义**上完全对应了计算机科学中“字典（Dictionary）”的概念，同时专门用于存储“资源（Resources）”。

`ResourceDictionary` 实现了 `IDictionary` 接口，其内部存储机制与标准的字典、哈希表完全一致：

- **Key（键）：** 每个资源必须有一个唯一的标识符，即 XAML 中的 `x:Key` 属性。
- **Value（值）：** 对应的实际对象（如 `Brush`, `Style`, `DataTemplate`, `string` 等）。

{{< /callout >}}

## 定义资源字典

资源字典通过 `<ResourceDictionary>` 元素定义，通常作为一个单独文件、或包含在 XAML 文件的 `<Window.Resources>` 或 `<Application.Resources>` 中。

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Style x:Key="ButtonStyle" TargetType="Button">
        <Setter Property="Background" Value="Black" />
        <Setter Property="Foreground" Value="White" />
    </Style>
</ResourceDictionary>
```

在这个例子中，将资源字典作为一个单独的文件存在，并在其中定义了一个按钮样式，并通过 `x:Key` 属性指定了唯一标识符。

> 若 Style 不设置 `x:Key`、仅指定 `TargetType`，会生成**隐式样式**，以控件类型作为隐式键；该样式仅在当前资源作用域内，自动匹配未手动指定 Style 的对应类型控件。

## 合并资源字典

可以通过 `MergedDictionaries` 属性将多个资源字典合并到一个资源字典中，然后只调用这一个资源字典，就可以将其余的资源字典一起引入，就像一个命名空间，便于模块化管理。

```xml
<ResourceDictionary>
    <ResourceDictionary.MergedDictionaries>
        <ResourceDictionary Source="Styles/Colors.xaml"/>
        <ResourceDictionary Source="Styles/Buttons.xaml"/>
    </ResourceDictionary.MergedDictionaries>
</ResourceDictionary>
```

在这个例子中，`Colors.xaml` 和 `Buttons.xaml` 是外部资源字典文件，分别定义了颜色和按钮样式。

## 使用资源字典

使用资源前，需要先在资源作用域中先注册声明，有这么一个资源存在，不然后续使用中会显示无此资源。

常用的资源作用域有三个：

1. Application 全局应用资源：位于 `App.xaml` 中的 `Application.Resources`
2. Window 窗口资源：位于窗口的 `Window.Resources`
3. 控件自身资源：单个控件的 `Resources`

> 还有一个父容器资源，比如在 Grid、StackPanel 等父面板的 `Resources` 中定义或声明了资源，其子控件都可以直接使用。

接下来，以全局应用资源为例，注册资源：

```xml
<Application xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Application.Resources>
		<ResourceDictionary Source="Styles/MainStyle.xaml" /> 
    </Application.Resources>
</Application>
```

如果要添加多个资源的话，就需要合并资源字典了。

然后就是正式的使用了。在对应的控件的 `Style` 属性上，通过 ` StaticResource ` 或 ` DynamicResource ` 引用样式。

```xml
<Button Content="Click Me" Style="{StaticResource ButtonStyle}" />
```

- `StaticResource`：XAML 加载阶段一次性查找解析资源，无法感知运行时资源变更，不支持向后引用资源，但性能更好；
- `DynamicResource`：延迟解析资源并监听资源变更，运行时替换、修改资源字典内容后界面自动刷新，支持前后任意顺序引用资源，有轻微性能开销，多用于主题切换、动态换色场景。