---
title: RSS 订阅源的生成与解析
slug: generating-and-parsing-rss-feeds
aliases:
  - /notes/generating-and-parsing-rss-feeds/
date: 2026-08-17T21:08:41+08:00
categories: 
  - 技术
  - 学习
tags:
  - csharp
  - 笔记
draft: false
description: "在本文中，记录了如何使用 C＃ System.ServiceModel.Syndication 进行 RSS、Atom 的生成与解析。"
---

## RSS 是什么

RSS（Really Simple Syndication，简易信息聚合）是一种基于 XML 规范的内容分发格式，用于将网站的文章、新闻、博客更新等内容封装为标准化的订阅源。

网站通过生成 RSS 源对外输出标题、摘要、发布时间、原文链接等元数据，然后客户端解析器读取该 XML 文档后，即可获得站点的更新内容，用户无需访问原始网页，就能获取最新资讯。

效果类似于订阅报纸，只要登记了报社的投递地址，每天就有新报纸自动送到家门口，不出家门便可知晓天下事。

RSS 实现了内容发布与内容阅读的分离，不依赖平台算法推送，兼顾隐私性与信息获取效率，广泛应用于博客、新闻站点、播客等场景，常见格式包含 RSS 2.0 以及功能更完善的 Atom 1.0 规范，二者是并列的两种标准。

## 在 C# 中使用什么库来处理 RSS

`System.ServiceModel.Syndication` 是 .NET 生态中的 Syndication（内容聚合）标准库，专门用于**生成与解析 RSS 2.0、Atom 1.0 格式的订阅源**。该库提供一套面向对象的模型，封装了订阅源、条目、作者、发布时间、摘要、链接等 RSS/Atom 的核心实体，开发者不需要手动拼接、解析原始 XML 字符串。

通过 `SyndicationFeed`、`SyndicationItem` 等核心类型，既可以将业务数据导出输出为标准 RSS 或 Atom XML 文档，也能够读取网络或本地的订阅源数据流，反序列化为内存对象，完成 RSS 源的解析读取。并且能够自动处理两种格式之间的格式差异，降低开发 RSS 相关功能的编码复杂度。

- `SyndicationFeed` 代表整个订阅源，在 RSS 2.0 中对应 `<channel>` 元素，在 Atom 1.0 中对应 `<feed>` 元素。
- `SyndicationItem` 代表订阅源里**单条资讯条目**，对应每一篇文章、新闻记录。

## 生成 RSS / Atom 格式的订阅源

使用 `System.ServiceModel.Syndication` 可以非常方便地在 .NET 中生成 RSS 2.0 和 Atom 1.0。该库的核心优势在于**模型与格式分离**：只需构建一个 `SyndicationFeed` 对象，然后通过不同的 `Formatter` 输出不同格式。

- **模型**：无论 RSS 还是 Atom，都通过 `SyndicationFeed` / `SyndicationItem` 操作
- **格式**：`Rss20FeedFormatter` 和 `Atom10FeedFormatter` 负责模型到 XML 的序列化

另外在使用之前，需要先安装该库，因为从 .NET 5 开始就不内置了。

```shell
dotnet add package System.ServiceModel.Syndication
```

### 构建模型

```csharp
using System.ServiceModel.Syndication;

// 创建订阅源对象
// 对应 RSS 2.0 中的 <channel> 或 Atom 中的 <feed>
var feed = new SyndicationFeed(
    title: "订阅源名称",                          // 订阅源标题
    description: "网站简介",                       // 订阅源描述
    feedAlternateLink: new Uri("https://example.com"), // 网站主页链接
    id: $"urn:uuid:{Guid.NewGuid()}",            // 订阅源的唯一标识符（URN 格式）
    lastUpdatedTime: DateTime.Now                // 最后更新时间
)
{
    Language = "zh-cn",                          // 订阅源的语言（中文简体）
    ImageUrl = new Uri("https://example.com/logo.png") // 订阅源图标/Logo 的 URL
};

// 添加订阅源的作者信息（支持多个作者）
feed.Authors.Add(new SyndicationPerson(
    email: "editor@example.com",   // 作者邮箱
    name: "名称",                   // 作者名称
    uri: "https://example.com"     // 作者主页链接
));

// 添加自定义链接，这里添加的是 Atom 规范中的 self link
// self link 指向订阅源文件本身的地址，方便订阅器识别源地址
feed.Links.Add(new SyndicationLink(
    uri: new Uri("https://example.com/feed.atom"), // 链接地址
    relationshipType: "self",                       // 关系类型为 self，表示这是订阅源自身
    title: "",                                      // 链接标题（此处为空）
    mediaType: "application/atom+xml",              // MIME 类型，表明是 Atom 格式
    length: 0                                       // 内容长度（未知时设为 0）
));

// 构建订阅条目列表
// 每个 SyndicationItem 对应一篇文章/一条资讯
var items = new List<SyndicationItem>
{
    new(
        title: "文章名称",                          // 条目标题
        content: new TextSyndicationContent(        // 条目正文内容
            "<p>正文 HTML 格式的内容...</p>",        // HTML 格式的正文
            TextSyndicationContentKind.Html          // 指定内容类型为 HTML
        ),
        itemAlternateLink: new Uri("https://example.com/posts/1"), // 文章原文链接
        id: $"urn:uuid:{Guid.NewGuid()}",           // 条目唯一标识符
        lastUpdatedTime: DateTime.Now               // 条目最后更新时间
    )
    {
        PublishDate = DateTime.Now,                 // 条目首次发布时间
        Summary = new TextSyndicationContent("文章摘要纯文本...") // 条目摘要（纯文本）
    }
};

// 将条目列表赋值给订阅源
feed.Items = items;
```

### 将模型序列化

#### 序列化为 RSS 2.0

```csharp
// 配置 XML 输出设置
var settings = new XmlWriterSettings
{
    Indent = true,                              // 启用缩进，使输出的 XML 格式美观可读
    Encoding = new UTF8Encoding(false),         // 使用 UTF-8 编码，false 表示不写入 BOM 头
};

// 创建内存流，用于接收序列化后的 XML 数据
using var stream = new MemoryStream();
// 创建 XmlWriter，将格式化后的 XML 写入内存流
using (var writer = XmlWriter.Create(stream, settings))
{
    // 使用 Rss20FeedFormatter 将 SyndicationFeed 模型序列化为 RSS 2.0 格式的 XML
    var rssFormatter = new Rss20FeedFormatter(feed);
    rssFormatter.WriteTo(writer);               // 执行写入操作
}

// 将内存流中的字节数组转换为 UTF-8 字符串，得到最终的 RSS XML 文本
var rssXml = Encoding.UTF8.GetString(stream.ToArray());
```

#### 序列化为 Atom 1.0

只需将 `Formatter` 替换为 `Atom10FeedFormatter`，即可输出 Atom 1.0 格式。

```csharp
var atomFormatter = new Atom10FeedFormatter(feed);
atomFormatter.WriteTo(writer);
```

#### 封装序列化过程中的通用逻辑

在输出不同格式之前，可以把通用的操作封装成一个单独的方法，然后传入想要的格式即可。

```csharp
/// <summary>
/// 将 SyndicationFeedFormatter 序列化为 XML 字符串。
/// 利用多态特性，传入 Rss20FeedFormatter 或 Atom10FeedFormatter 均可正常工作。
/// </summary>
/// <param name="formatter">
/// 订阅源格式化器，SyndicationFeedFormatter 是 Rss20FeedFormatter 
/// 和 Atom10FeedFormatter 的共同基类
/// </param>
/// <returns>序列化后的 XML 字符串</returns>
public string FormatFeed(SyndicationFeedFormatter formatter)
{
    // 配置 XML 写入设置
    var settings = new XmlWriterSettings
    {
        Indent = true,
        Encoding = new UTF8Encoding(false), // 不带 BOM
    };

    // 使用内存流作为写入目标
    using var stream = new MemoryStream();
    // 通过 XmlWriter 将格式化器内容写入流
    using (var writer = XmlWriter.Create(stream, settings))
    {
        formatter.WriteTo(writer);
    }

    // 将流内容转为字符串并返回
    return Encoding.UTF8.GetString(stream.ToArray());
}
```

```csharp
// 分别输出为 RSS 2.0 和 Atom 1.0
var rssOutput = FormatFeed(new Rss20FeedFormatter(feed));
var atomOutput = FormatFeed(new Atom10FeedFormatter(feed));
```

在 `FormatFeed` 方法中，`SyndicationFeedFormatter` 类是 `Rss20FeedFormatter` 和 `Atom10FeedFormatter` 共同的基类，也就是说，只需要传入不同的派生子类，就可以无需关心具体实现，也是利用了面向对象中的多态特性。

### 输出结果

**RSS 2.0**：

```xml
<?xml version="1.0" encoding="utf-8"?>
<rss xmlns:a10="http://www.w3.org/2005/Atom" version="2.0">
  <channel>
    <title>订阅源名称</title>
    <link>https://example.com/</link>
    <description>网站简介</description>
    <language>zh-cn</language>
    <managingEditor>editor@example.com</managingEditor>
    <lastBuildDate>Wed, 05 Aug 2026 14:46:52 +0800</lastBuildDate>
    <image>
      <url>https://example.com/logo.png</url>
      <title>订阅源名称</title>
      <link>https://example.com/</link>
    </image>
    <a10:id>urn:uuid:75551e8c-cefa-4510-a6ff-335614da5435</a10:id>
    <a10:link rel="self" type="application/atom+xml" href="https://example.com/feed.atom" />
    <item>
      <guid isPermaLink="false">urn:uuid:0e684b46-1dbf-4f69-9db3-496fdad6b32e</guid>
      <link>https://example.com/posts/1</link>
      <title>文章名称</title>
      <description>文章摘要纯文本...</description>
      <pubDate>Wed, 05 Aug 2026 14:46:52 +0800</pubDate>
      <a10:updated>2026-08-05T14:46:52+08:00</a10:updated>
      <a10:content type="html">&lt;p&gt;文章内容&lt;/p&gt;</a10:content>
    </item>
  </channel>
</rss>
```

**Atom 1.0**：

```xml
<?xml version="1.0" encoding="utf-8"?>
<feed xml:lang="zh-cn" xmlns="http://www.w3.org/2005/Atom">
  <title type="text">订阅源名称</title>
  <subtitle type="text">网站简介</subtitle>
  <id>urn:uuid:75551e8c-cefa-4510-a6ff-335614da5435</id>
  <updated>2026-08-05T14:46:52+08:00</updated>
  <logo>https://example.com/logo.png</logo>
  <author>
    <name>名称</name>
    <uri>https://example.com</uri>
    <email>editor@example.com</email>
  </author>
  <link rel="alternate" href="https://example.com/" />
  <link rel="self" type="application/atom+xml" href="https://example.com/feed.atom" />
  <entry>
    <id>urn:uuid:0e684b46-1dbf-4f69-9db3-496fdad6b32e</id>
    <title type="text">文章名称</title>
    <summary type="text">文章摘要纯文本...</summary>
    <published>2026-08-05T14:46:52+08:00</published>
    <updated>2026-08-05T14:46:52+08:00</updated>
    <link rel="alternate" href="https://example.com/posts/1" />
    <content type="html">&lt;p&gt;文章内容&lt;/p&gt;</content>
  </entry>
</feed>
```

## 解析 RSS / Atom 订阅源

`System.ServiceModel.Syndication` 提供了统一的解析模型：无论输入是 RSS 2.0 还是 Atom 1.0，都通过 `SyndicationFeed.Load()` 自动识别并映射到同一个 `SyndicationFeed` 对象。

```csharp
// 创建 XmlReader 读取远程订阅源（支持 RSS 2.0 和 Atom 1.0，自动识别格式）
using var reader = XmlReader.Create("https://example.com/feed");

// 使用 SyndicationFeed.Load() 解析订阅源，返回统一的 SyndicationFeed 对象
// 无论原始格式是 RSS 还是 Atom，都会映射到相同的对象模型
SyndicationFeed feed = SyndicationFeed.Load(reader);
```

```csharp
Console.WriteLine($"标题: {feed.Title?.Text}");
Console.WriteLine($"描述: {feed.Description?.Text}");
Console.WriteLine($"最后更新: {feed.LastUpdatedTime}");
Console.WriteLine($"作者: {string.Join(", ", feed.Authors.Select(a => a.Name))}");
Console.WriteLine($"链接数: {feed.Links.Count}");
Console.WriteLine($"条目数: {feed.Items.Count()}");
Console.WriteLine();

foreach (SyndicationItem item in feed.Items)
{
    Console.WriteLine($"--- 条目 ---");
    Console.WriteLine($"  标题: {item.Title?.Text}");
    Console.WriteLine($"  ID: {item.Id}");
    Console.WriteLine($"  发布: {item.PublishDate}");
    Console.WriteLine($"  更新: {item.LastUpdatedTime}");
    Console.WriteLine($"  链接: {item.Links.FirstOrDefault()?.Uri}");
    Console.WriteLine($"  摘要: {item.Summary?.Text}");

    // 区分内容类型
    if (item.Content is TextSyndicationContent textContent)
    {
        Console.WriteLine($"  内容类型: {textContent.Type}");
        Console.WriteLine($"  内容: {textContent.Text}");
    }
    else if (item.Content is UrlSyndicationContent urlContent)
    {
        Console.WriteLine($"  媒体URL: {urlContent.Url} ({urlContent.MediaType})");
    }

    Console.WriteLine();
}
```

## 参考链接

- [WCF Syndication Overview - WCF | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/framework/wcf/feature-details/wcf-syndication-overview)
- [How to: Create a Basic Atom Feed - WCF | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/framework/wcf/feature-details/how-to-create-a-basic-atom-feed)
- [How to: Expose a Feed as Both Atom and RSS - WCF | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/framework/wcf/feature-details/how-to-expose-a-feed-as-both-atom-and-rss)