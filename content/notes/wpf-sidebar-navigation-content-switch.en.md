---
title: Implement Content Switching via Side Navigation Bar in WPF
date: 2026-03-08T15:46:41+08:00
tags:
  - Note
  - CSharp
  - Wpf
draft: false
description: "Introduces how to implement a sidebar navigation layout in WPF using ListBox, ContentControl, and UserControl, enabling content switching to display different page views."
summary: "This article documents two WPF sidebar navigation implementation approaches: Event-Driven pattern and MVVM pattern. The Event-Driven approach triggers content switching through the ListBox's SelectionChanged event; the MVVM approach uses the CommunityToolkit.Mvvm framework, achieving dynamic association between navigation items and views through data binding, and simplifying view instantiation using reflection."
---

This note mainly documents how to use three controls (ListBox + ContentControl + UserControl) in WPF to implement content switching in the form of a side navigation bar.

The effect diagram is as follows:

```txt
┌─────────┬────────────────────────────┐
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

The development environment used is as follows:

- Language version: .NET 10
- MVVM framework: CommunityToolkit.Mvvm 8.4.0

## Event-driven Implementation Solution

The core implementation idea is to use the ListBox control to build the navigation bar, then create a selection event. When the event is triggered, assign the corresponding UserControl to the Content property of the ContentControl, thereby achieving content switching.

- XAML Code

```xml {filename="MainWindow.xaml"}
<DockPanel>
    <!-- Navigation bar list box for page navigation switching, including three options: Home, Settings, About -->
    <ListBox x:Name="NavBar" SelectionChanged="NavBar_OnSelectionChanged">
        <ListBoxItem>Home</ListBoxItem>
        <ListBoxItem>Settings</ListBoxItem>
        <ListBoxItem>About</ListBoxItem>
    </ListBox>
    <!-- Content control for displaying page content corresponding to navigation options -->
    <ContentControl x:Name="ContCtrl"></ContentControl>
</DockPanel>
```

- Code-behind

```csharp {filename="MainWindow.xaml.cs"}
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        // Initialize the navigation bar, select the first item by default, and set the content control to display the home view
        NavBar.SelectedIndex = 0;
        ContCtrl.Content = new HomeView();
    }

    // Event handler triggered when the selected item of the navigation bar changes
    private void NavBar_OnSelectionChanged(object sender, SelectionChangedEventArgs e)
    {
        // Get the currently selected list item
        var item = NavBar.SelectedItem as ListBoxItem;

        // Switch to the corresponding view page based on the content of the selected item
        ContCtrl.Content = item?.Content switch
        {
            "Home" => new HomeView(),
            "Settings" => new SettingView(),
            "About" => new AboutView(),
            _ => ContCtrl
        };
    }
}
```

- Custom UserControl Content. Since the implementation is simple at this stage, the three user controls only differ in text content, so only the content of one UserControl is provided here.

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
        <TextBlock Text="Home"
                   HorizontalAlignment="Center" 
                   VerticalAlignment="Center" 
                   FontSize="20"/>
    </Grid>
</UserControl>
```

## MVVM Pattern Implementation Solution

The implementation idea is not significantly different from the event-driven approach. It still detects whether the selected item of the ListBox has changed; if it does, it modifies the Content property of the ContentControl to achieve dynamic switching.

Here, the event-triggered logic is replaced with data binding. When the value of the bound property changes, the change method is invoked.

Additionally, several enhancements are made, such as:

- Customizing the ListBox template to display personalized content, and navigation items are added from the backend to enable dynamic updates.
- The data for navigation items is replaced with a custom record structure, which defines the content to be displayed and stores the corresponding view type for switching, simplifying the view instantiation process using reflection.
- Using the [CommunityToolkit.Mvvm](https://github.com/CommunityToolkit/dotnet) framework to simplify the implementation of the MVVM pattern.

The code implementation is as follows:

- XAML Code

```xml {filename="MainWindow.xaml"}
<DockPanel>
    <!-- Navigation bar list box, bound to the NavItems collection and SelectedItem property of the ViewModel
         to achieve data-driven display of navigation items -->
    <ListBox x:Name="NavBar" ItemsSource="{Binding NavItems}"
             SelectedItem="{Binding SelectedItem}">
        <ListBox.ItemTemplate>
            <!-- Custom list item template, defining the display layout of each navigation item -->
            <DataTemplate>
                <DockPanel>
                    <!-- Display the icon of the navigation item -->
                    <TextBlock Text="{Binding Icon}"/>
                    <!-- Display the title text of the navigation item -->
                    <TextBlock Text="{Binding Title}"/>
                </DockPanel>
            </DataTemplate>
        </ListBox.ItemTemplate>
    </ListBox>
    <!-- Content control, bound to the CurrentView property of the ViewModel to dynamically display the currently selected view -->
    <ContentControl Content="{Binding CurrentView}"></ContentControl>
</DockPanel>
```

- Custom ViewModel

```csharp {filename="MainWindowViewModel.cs"}
public partial class MainWindowViewModel : ObservableObject
{
    // Property for the selected navigation item
    [ObservableProperty]
    private NavItem _selectedItem;
    
    // Property for the currently displayed view object
    [ObservableProperty]
    private object _currentView;
    
    // Array of navigation bar items, containing all available navigation options
    public NavItem[] NavItems { get; init; }
    
    public MainWindowViewModel()
    {
        // Initialize three navigation items: Home, Settings, About, each associated with the corresponding view type
        NavItems =
        [
            new NavItem("Home", "🏠️", typeof(HomeView)),
            new NavItem("Settings", "⚙️", typeof(SettingView)),
            new NavItem("About", "ℹ️", typeof(AboutView))
        ];
        // Select the first navigation item (Home) by default
        SelectedItem = NavItems[0];
    }

    /// <summary>
    /// Method implementation when the SelectedItem property changes, used to switch the currently displayed view
    /// </summary>
    /// <param name="value">The newly selected navigation item, containing type information of the target view</param>
    partial void OnSelectedItemChanged(NavItem value)
    {
        // Create an instance of the corresponding view type through reflection
        var view = Activator.CreateInstance(value.View);
        // Update the current view if creation is successful; otherwise, keep the original view
        CurrentView = view ?? CurrentView;
    }
}
```

- Navigation Item Type Definition

```csharp {filename="NavItem.cs"}
/// <summary>
/// Navigation item record type, encapsulating display information of navigation bar items and associated view types
/// </summary>
/// <param name="Title">Display title text of the navigation item</param>
/// <param name="Icon">Icon character displayed by the navigation item</param>
/// <param name="View">View type associated with the navigation item, used to create the corresponding page</param>
public record NavItem(string Title, string Icon, Type View);
```

- UserControl remains unchanged

## Summary

The reason for choosing ListBox as the navigation bar is that it has a single-selection feature, which can easily achieve option exclusivity and ensure only one option is selected at a time.

Initially, I intended to implement it using Button + layout controls, but found that implementing exclusivity was rather cumbersome, so I switched to using the ListBox control instead.