---
title: 关于WPF的一些小设置
date: 2026-05-11T20:55:56+08:00
tags:
  - 笔记
  - csharp
  - wpf
draft: false
description: 这里收集了一些关于WPF有用的小设置，目前包括窗口无边框、Datacontext设置。
summary: 这里收集了一些关于WPF有用的小设置，目前包括窗口无边框、Datacontext设置。
---


# 无边框（`WindowChrome`）

[WindowChrome](https://learn.microsoft.com/dotnet/api/system.windows.shell.windowchrome) 是微软官方推荐的**无边框窗口实现方案**，在保留窗口所有原生功能（拖动、调整大小、最小化最大化关闭）的情况下，只隐藏或移除传统的标题栏。

使用方式也很简单，如下代码所示：

```xml
<Window ...>
    <WindowChrome.WindowChrome>
        <WindowChrome CaptionHeight="50"
                      UseAeroCaptionButtons="False"/>
    </WindowChrome.WindowChrome>
</Window>
```

- `CaptionHeight`：定义标题栏（非客户区）的高度，该区域内的拖动操作会被识别为窗口拖动
- `UseAeroCaptionButtons`：决定是否使用系统默认的 Aero 风格标题栏按钮。设为 False 后可使用自定义按钮

**示例**：

```xml
<Window ...
        WindowStartupLocation="CenterScreen">
    <!-- 无边框设置 -->
    <WindowChrome.WindowChrome>
        <WindowChrome CaptionHeight="50"
                      UseAeroCaptionButtons="False" />
    </WindowChrome.WindowChrome>

    <Grid RowDefinitions="Auto, *">
        <!-- 自定义标题栏（高度和 CaptionHeight 一致） -->
        <DockPanel Grid.Row="0" 
                   Height="50" 
                   Background="AliceBlue" 
                   LastChildFill="False">
            <TextBlock DockPanel.Dock="Left" 
                       Text="自定义标题栏" 
                       VerticalAlignment="Center" 
                       Margin="28,0" />

            <!-- 最小化、最大化、关闭按钮 -->
            <StackPanel DockPanel.Dock="Right" 
                        Orientation="Horizontal"
                        WindowChrome.IsHitTestVisibleInChrome="True">
                <Button x:Name="BtnMin" 
                        Content="—" 
                        Width="50"  
                        BorderThickness="0"
                        Background="Transparent" 
                        FontSize="16" />
                <Button x:Name="BtnMax" 
                        Content="□" 
                        Width="50"  
                        BorderThickness="0"
                        Background="Transparent" FontSize="16" />
                <Button x:Name="BtnClose" 
                        Content="×" 
                        Width="50"  
                        BorderThickness="0"
                        Background="Transparent" 
                        FontSize="18" />
            </StackPanel>
        </DockPanel>
        <!-- 主内容区 -->
        <Grid Grid.Row="1">
            <TextBlock Text="主内容区域" 
                       HorizontalAlignment="Center" 
                       VerticalAlignment="Center" 
                       FontSize="18" />
        </Grid>
    </Grid>
</Window>
```

```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        BtnMin.Click += (s, e) => WindowState = WindowState.Minimized;
        BtnMax.Click += 
            (s, e) => WindowState = 
                WindowState == WindowState.Maximized
                                ? WindowState.Normal
                                : WindowState.Maximized;
        BtnClose.Click += (s, e) => Close();
    }
}
```

在 `CaptionHeight` 划定的区域内，控件无法被响应鼠标的点击，这是因为标题栏区域位于它们的上层，挡住了这些控件，因此若想让这些控件可被点击，需要添加 `WindowChrome.IsHitTestVisibleInChrome` 属性。

```xml
<Button WindowChrome.IsHitTestVisibleInChrome="True"></Button>
```

`StackPanel` 加上了这个属性，其中的 `Button` 也可以点击了，在这个地方省点力，不用一个个添加了。

```xml {hl_lines=[4]}
<!-- 最小化、最大化、关闭按钮 -->
<StackPanel DockPanel.Dock="Right" 
            Orientation="Horizontal"
            WindowChrome.IsHitTestVisibleInChrome="True">
    ...
</StackPanel>
```

# 设置 `DataContext`

`DataContext` 是每个 WPF 控件都拥有的一个属性，它代表了该控件的 "**默认数据源**" 或 "**数据上下文**"。

如果一个控件没有显式设置自己的 `DataContext`，它会自动继承其父控件的 `DataContext`。

在这里，主要演示 Window 的 `DataContext`。

## 在后置代码中设置

```csharp {hl_lines=[4]}
public MainWindow()
{
    InitializeComponent();
    this.DataContext = new Something();
}
```

## 在 XAML 中设置

在界面中设置 `DataContext`，需要先引入数据源所在的命名空间。

比如说，我们的数据源是 `Something` 类，定义在 `Example.Models` 命名空间中。

```csharp
namespace Example.Models;

public class Something
{

}
```

然后在 XAML 中引入这个命名空间，就可以直接在 `Window.DataContext` 声明要使用的数据源。

```xml {hl_lines=[2,5]}
<Window ...
        xmlns:model="clr-namespace:Example.Models">
        
	<Window.DataContext>
	    <model:Something/>
	</Window.DataContext>
</Window>
```

## 绑定设计时数据

使用设计时 `DataContext`，可以在编辑 XAML 时，设计器里直接看到绑定数据、预览界面效果，并且不影响运行时真正的数据绑定。

```xml {hl_lines=[5]}
<Window ...
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:model="clr-namespace:Example.Models"
        mc:Ignorable="d" 
        d:DataContext="{d:DesignInstance model:Something, IsDesignTimeCreatable=True}">
	...
</Window>
```

- `DesignInstance`：指定设计时数据上下文类型
- `IsDesignTimeCreatable`：是一个布尔属性，用于控制设计器如何处理 `DesignInstance`