---
title: 在 WPF 中通过侧边导航栏实现内容切换
date: 2026-03-08T15:46:30+08:00
tags:
  - 笔记
  - CSharp
  - Wpf
draft: false
description: 介绍如何在 WPF 中使用 ListBox、ContentControl 和 UserControl 实现侧边导航栏布局，通过内容切换展示不同页面视图。
summary: 本文记录了两种 WPF 侧边导航栏实现方案：事件驱动模式和 MVVM 模式。事件驱动方案通过 ListBox 的 SelectionChanged 事件触发内容切换；MVVM 方案使用 CommunityToolkit.Mvvm 框架，通过数据绑定实现导航项与视图的动态关联，并利用反射简化视图实例化过程。
---

本篇笔记主要记录，如何在 WPF 中利用 ListBox+ContentControl+UserControl 三个控件，以侧边导航栏的形式实现内容切换。

效果图如下：

```txt
┌─────────┬────────────────────────────┐
│  Title  │                            │
├─────────┤                            │
│  Title  │                            │
├─────────┤                            │
│  Title  │                            │
├─────────┤           Content          │
│         │                            │
│         │                            │
│         │                            │
│         │                            │
│         │                            │
└─────────┴────────────────────────────┘
```

所使用的开发环境如下：

- 语言版本：.Net 10
- MVVM 框架：CommunityToolkit.Mvvm 8.4.0

## 以事件驱动的实现方案

具体实现思路为，利用 ListBox 控件搭建导航栏，然后创建选中事件，如果事件被触发，则将对应的 UserControl 控件赋值给 ContentControl 控件的 Content 属性，至此实现内容切换。

- XAML 代码

```xml {filename="MainWindow.xaml"}
<DockPanel>
    <!-- 导航栏列表框，用于页面导航切换，包含首页、设置、关于三个选项 -->
    <ListBox x:Name="NavBar" SelectionChanged="NavBar_OnSelectionChanged">
        <ListBoxItem>首页</ListBoxItem>
        <ListBoxItem>设置</ListBoxItem>
        <ListBoxItem>关于</ListBoxItem>
    </ListBox>
    <!-- 内容控件，用于显示与导航选项对应的页面内容 -->
    <ContentControl x:Name="ContCtrl"></ContentControl>
</DockPanel>
```

- 后置代码

```csharp {filename="MainWindow.xaml.cs"}
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        // 初始化导航栏，默认选中第一项，并设置内容控件显示首页视图
        NavBar.SelectedIndex = 0;
        ContCtrl.Content = new HomeView();
    }

    // 当导航栏选择项发生改变时，触发的事件处理函数
    private void NavBar_OnSelectionChanged(object sender, SelectionChangedEventArgs e)
    {
        // 获取当前选中的列表项
        var item = NavBar.SelectedItem as ListBoxItem;

        // 根据选中项的内容切换到对应的视图页面
        ContCtrl.Content = item?.Content switch
        {
            "首页" => new HomeView(),
            "设置" => new SettingView(),
            "关于" => new AboutView(),
            _ => ContCtrl
        };
    }
}
```

- 自定义 UserControl 控件内容。由于现阶段实现的很简单，三个用户控件只有文本不同，所以在这里只给出了其中一个 UserControl 控件的内容

```xml {filename="HomeView.xaml", hl_lines=11}
<UserControl x:Class="TryDemo.HomeView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:local="clr-namespace:TryDemo"
             mc:Ignorable="d"
             d:DesignHeight="300" d:DesignWidth="300">
    
    <Grid Background="LightBlue">
        <TextBlock Text="首页"
                   HorizontalAlignment="Center" 
                   VerticalAlignment="Center" 
                   FontSize="20"/>
    </Grid>
</UserControl>
```

## MVVM 模式的实现方案

与事件驱动的实现思路，并没有多大的区别，依旧是检测 ListBox 选择项是否更改，如果发生变化，则修改 ContentControl 的 Content 属性，实现动态切换。

在这里，将事件触发的逻辑改为了数据绑定，当被绑定的属性值发生变化，会调用更改方法。

同时也进行了部分升级，比如：

- 自定义 ListBox 模板，显示个性化内容，同时导航项也变为从后台添加，依此可以实现动态更新。
- 关于导航项的数据，则变为了自定义的记录结构，其中定义了要显示的内容，同时保存了对应要切换的视图类型，利用反射简化视图实例化过程。
- 使用 [CommunityToolkit.Mvvm](https://github.com/CommunityToolkit/dotnet) 框架，简化 MVVM 模式的实现。

下面就是代码实现了：

- XAML 代码

```xml {filename="MainWindow.xaml"}
<DockPanel>
    <!-- 导航栏列表框，绑定到 ViewModel 的 NavItems 集合和 SelectedItem 属性
         ，实现导航项的数据驱动显示 -->
    <ListBox x:Name="NavBar" ItemsSource="{Binding NavItems}"
             SelectedItem="{Binding SelectedItem}">
        <ListBox.ItemTemplate>
            <!-- 自定义列表项模板，定义每个导航项的显示布局 -->
            <DataTemplate>
                <DockPanel>
                    <!-- 显示导航项的图标 -->
                    <TextBlock Text="{Binding Icon}"/>
                    <!-- 显示导航项的标题文本 -->
                    <TextBlock Text="{Binding Title}"/>
                </DockPanel>
            </DataTemplate>
        </ListBox.ItemTemplate>
    </ListBox>
    <!-- 内容控制器，绑定到 ViewModel 的 CurrentView 属性，动态显示当前选中的视图 -->
    <ContentControl Content="{Binding CurrentView}"></ContentControl>
</DockPanel>
```

- 自定义的视图模型

```csharp {filename="MainWindowViewModel.cs"}
public partial class MainWindowViewModel : ObservableObject
{
    // 被选中的导航项属性
    [ObservableProperty]
    private NavItem _selectedItem;
    
    // 当前显示的视图对象属性
    [ObservableProperty]
    private object _currentView;
    
    // 导航栏项目数组，包含所有可用的导航选项
    public NavItem[] NavItems { get; init; }
    
    public MainWindowViewModel()
    {
        // 初始化三个导航项：首页、设置、关于，分别关联对应的视图类型
        NavItems =
        [
            new NavItem("首页", "🏠️", typeof(HomeView)),
            new NavItem("设置", "⚙️", typeof(SettingView)),
            new NavItem("关于", "ℹ️", typeof(AboutView))
        ];
        // 默认选中第一个导航项（首页）
        SelectedItem = NavItems[0];
    }

    /// <summary>
    /// SelectedItem 属性改变时的方法实现，用于切换当前显示的视图
    /// </summary>
    /// <param name="value">新选中的导航项，包含目标视图的类型信息</param>
    partial void OnSelectedItemChanged(NavItem value)
    {
        // 通过反射创建对应视图类型的实例
        var view = Activator.CreateInstance(value.View);
        // 如果创建成功则更新当前视图，否则保持原视图不变
        CurrentView = view ?? CurrentView;
    }
}
```

- 导航项类型定义

```csharp {filename="NavItem.cs"}
/// <summary>
/// 导航项记录类型，封装导航栏项目的显示信息和关联的视图类型
/// </summary>
/// <param name="Title">导航项显示的标题文本</param>
/// <param name="Icon">导航项显示的图标字符</param>
/// <param name="View">导航项关联的视图类型，用于创建对应的页面</param>
public record NavItem(string Title, string Icon, Type View);
```

- UserControl 控件保持不变

## 小结

在这里选择 ListBox 作为导航栏的原因是，其具有单选功能，可以轻易地实现选项排他性，保证只有一个选项被选中。

一开始想的是用 Button + 布局控件实现，但发现排他性的实现比较麻烦，转而使用 ListBox 控件。
