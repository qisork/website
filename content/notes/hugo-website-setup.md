---
title: 使用 HUGO 搭建网站
date: 2025-06-22T09:54:23+08:00
tags:
  - hugo
  - static-site
  - 教程
draft: false
description: 本篇文章将介绍如何使用 Hugo 搭建一个静态网站，并配置 PaperMod 主题。
summary: 本篇文章将介绍如何使用 Hugo 搭建一个静态网站，并配置 PaperMod 主题。
---

## 安装 Hugo 并创建网站

### 安装 Hugo

首先，确保电脑上已安装 Hugo。可以前往 [Hugo 官网](https://gohugo.io/installation/) 下载适用于您操作系统的版本。安装完成后，在终端中输入以下命令验证是否安装成功：

```bash
hugo version
```

### 创建新项目

首先，我们需要创建一个新的 Hugo 网站项目。请打开终端（或命令行工具），依次输入以下命令：

```shell
hugo new site quickstart
cd quickstart
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

- `hugo new site quickstart`：创建一个名为 `quickstart` 的新 Hugo 项目。
- `cd quickstart`：进入该项目目录。
- `git init`：初始化 Git 版本控制系统，便于后续管理代码。
- `git submodule add ...`：将 PaperMod 主题作为子模块添加到 `themes/` 目录中。

> **注意：** 如果网络不稳定，下载可能会失败，请确保网络畅通或尝试更换镜像源。

### 指定主题

接下来，我们告诉 Hugo 使用哪个主题来渲染网站。编辑项目根目录下的 `hugo.toml` 文件，添加如下内容：

```toml
theme = "PaperMod"
```

### 预览网站

完成以上步骤后，运行以下命令启动本地开发服务器：

```shell
hugo server
```

此时，可以在浏览器中访问 `http://localhost:1313/` 预览网站。按下 `Ctrl + C` 即可停止服务。

## 配置 Hugo 网站

### 基础配置

Hugo 的核心配置文件是 `hugo.toml`，它决定了网站的基本行为。我们可以根据需要对其进行修改。

{{% details title="这里是完整配置" closed="true" %}}

```toml
# 网站的基础全局配置
baseURL                = "https://example.org/" # 设置网站的根 URL，用于生成绝对链接，部署时应更改为实际域名
languageCode           = "zh"                   # 指定网站内容语言编码，用于 HTML lang 属性及语言相关的设置
title                  = "ExampleSite"          # 网站主标题，通常显示在浏览器标签页和首页
theme                  = "PaperMod"             # 使用的主题名称，决定了网站外观风格

# 网站图标的资源配置，用于定义不同尺寸和用途的网站图标
[assets]
  # 图标放在 static/ 目录下
  favicon          = "/favicon.ico"          # 主要 favicon 图标，通常显示在浏览器标签页上
  favicon16x16     = "/favicon-16x16.png"    # 16x16 像素大小的 favicon，适用于低分辨率设备
  favicon32x32     = "/favicon-32x32.png"    # 32x32 像素大小的 favicon，提供更高清显示
  apple_touch_icon = "/apple-touch-icon.png" # Apple 设备上添加到主屏幕时使用的图标

# 主题参数配置，影响网站外观与功能行为
[params]
  # SEO 相关设置，有助于搜索引擎优化
  title       = "ExampleSite"             # 设置网站标题，显示在浏览器标签和搜索引擎结果中
  description = "ExampleSite description" # 网站的简要描述，用于搜索引擎摘要展示
  keywords    = ["Blog", "Portfolio"]     # 定义网站相关的关键词列表，提升搜索相关性

  # 基础网站行为配置
  DateFormat   = "2006-01-02" # 设置日期格式化样式，用于文章发布时间等时间字段的显示
  defaultTheme = "auto"       # 设置默认主题模式："auto" 表示跟随系统明暗设置，也可以设为 "light" 或 "dark"

  # 首页欢迎信息
  [params.homeInfoParams]
    Title   = "Hi there \U0001F44B"
    Content = "Welcome to my blog"

# 全局导航菜单配置，适用于所有语言环境下的主菜单项
[menu]
  [[menu.main]] # 定义一个主菜单项
    identifier = "posts"   # 菜单项标识符
    name       = "文章"      # 菜单项名称
    url        = "/posts/" # 菜单项链接
    weight     = 1         # 菜单项权重，用于排序
  [[menu.main]]
    identifier = "tags"
    name       = "标签"
    url        = "/tags/"
    weight     = 2
  [[menu.main]]
    identifier = "search"
    name       = "搜索"
    url        = "/search/"
    weight     = 3

# 输出格式设置
[outputs]
  # 为搜索功能提供支持
  home = ["HTML", "RSS", "JSON"]
```

{{% /details %}}

#### 网站基本信息设置

```toml
baseURL                = "https://example.org/"
languageCode           = "zh"
title                  = "ExampleSite"
theme                  = "PaperMod"
```

这些配置项定义了网站的基本信息，例如标题、语言和主题等。其中：

- `baseURL` 是部署时使用的域名。
- `languageCode` 控制网站的语言环境。
- `title` 显示在浏览器标签页上。
- `theme` 指定要使用的主题。

#### 图标设置

为了让网站更具个性化，我们可以为它设置图标。将以下内容添加至 `hugo.toml`：

```toml
[assets]
  favicon          = "/favicon.ico"
  favicon16x16     = "/favicon-16x16.png"
  favicon32x32     = "/favicon-32x32.png"
  apple_touch_icon = "/apple-touch-icon.png"
```

这些图标应放置在 `static/` 目录下，用于不同设备和浏览器显示。

#### SEO 设置

搜索引擎优化（SEO）对于提升网站曝光率至关重要。我们可以为网站添加如下配置：

```toml
[params]
  title       = "ExampleSite"
  description = "ExampleSite description"
  keywords    = ["Blog", "Portfolio"]
```

- `title`：网站标题，显示在搜索引擎结果中。
- `description`：简要描述，用于摘要展示。
- `keywords`：关键词列表，有助于提高搜索排名。

### 导航菜单配置

导航栏是用户浏览网站的重要入口。我们可以通过如下方式定义菜单项：

```toml
[menu]
  [[menu.main]]
    identifier = "posts"
    name       = "文章"
    url        = "/posts/"
    weight     = 1
  [[menu.main]]
    identifier = "tags"
    name       = "标签"
    url        = "/tags/"
    weight     = 2
  [[menu.main]]
    identifier = "search"
    name       = "搜索"
    url        = "/search/"
    weight     = 3
```

以上配置会生成三个菜单项：“文章”、“标签”和“搜索”，并按权重排序。

- **identifier**：给这个菜单项起个唯一的名字（ID），方便程序识别和使用。
- **name**：这是显示在网页上的名字，用户会看到“文章”这两个字。
- **url**：点击这个菜单项后跳转的网址，这里是网站内的“文章列表页”。
- **weight**：设置排序顺序，数字越小排得越靠前。

### 搜索页面创建

现在”搜索“界面还是处于 404 状态中，因为 PaperMod 只内置了标签页面，无需手动创建。

为了避免这种情况，我们需要在 `content/` 目录下，新建 `search.md` 文件，并添加如下配置：

- `search.md`

```yaml
---
title: "搜索"
layout: "search"
summary: "搜索页面"
placeholder: "这里可以输入..."
---
```

- **title**：设置该页面的标题。
- **layout**：指定该页面使用的布局模板。
- **summary**：提供该页面的简要描述。
- **placeholder**：设置页面中搜索框的占位符文本。

### 输出格式设置

为了支持搜索功能，我们需要为首页指定额外的输出格式：

```toml
[outputs]
  home = ["HTML", "RSS", "JSON"]
```

这样，首页不仅会生成 HTML 页面，还会提供 RSS 订阅和 JSON 数据接口。

---

到这里，网站已经配置完成，可以上传到诸如 GitHub Pages、Cloudflare Pages、Netlify 等平台进行部署，详细过程请参考 [Host and deploy | HUGO](https://gohugo.io/host-and-deploy/)。

## 可选：配置多语言支持

### 开启多语言支持

要启用多语言功能，首先需要修改 Hugo 的全局配置文件 `hugo.toml`，将原本的基础菜单和语言设置替换为以下结构化的多语言配置：

```toml
[languages]
  [languages.zh] # 中文语言配置
    languageCode = "zh"
    languageName = "中文"
    weight       = 1

    [languages.zh.menu]
      [[languages.zh.menu.main]]
        identifier = "posts"
        name       = "文章"
        url        = "/posts/"
        weight     = 1
      [[languages.zh.menu.main]]
        identifier = "tags"
        name       = "标签"
        url        = "/tags/"
        weight     = 2
      [[languages.zh.menu.main]]
        identifier = "search"
        name       = "搜索"
        url        = "/search/"
        weight     = 3

  [languages.en] # 英文语言配置
    languageCode = "en"
    languageName = "English"
    weight       = 2

    [languages.en.menu]
      [[languages.en.menu.main]]
        identifier = "posts"
        name       = "Posts"
        url        = "/posts/"
        weight     = 1
      [[languages.en.menu.main]]
        identifier = "tags"
        name       = "Tags"
        url        = "/tags/"
        weight     = 2
      [[languages.en.menu.main]]
        identifier = "search"
        name       = "Search"
        url        = "/search/"
        weight     = 3
```

- `[languages]` 是 Hugo 的多语言主配置块。
- 每个子项（如 `languages.zh` 和 `languages.en`）代表一种语言。
- `languageCode` 用于识别语言类型，通常使用 ISO 标准代码（如 `"zh"` 表示中文，`"en"` 表示英文）。
- `languageName` 是显示在网页上的语言名称。
- `weight` 控制语言切换顺序，数值越小优先级越高。

> **提示：** 如果有更多语言需求（如法语、德语等），只需按照上述格式继续添加即可。

### 创建多语言页面文件

为了支持多语言，我们需要为每种语言单独创建对应的页面文件。

根据 [Multilingual mode | HUGO](https://gohugo.io/content-management/multilingual/#translate-your-content) 官网文档描述，可以通过两种方法创建，并管理日后的文章：

1. 使用文件名区分
	- `/content/search.zh.md`
	- `/content/search.en.md`
2. 使用目录区分
	- `/content/zh/search.md`
	- `/content/en/search.md`

选择哪种方式创建，取决于个人喜好。这里我们选择“目录”形式，而使用文件名则无需配置，直接将对应语言的文件名添加相应的语言代码即可，所以对目录形式的配置进行着重讲解。

进入 `content/` 目录，按如下方式组织结构：

```
content/
├── zh/
│   └── search.md    # 中文版搜索页
└── en/
    └── search.md    # 英文版搜索页
```

分别编辑对应语言的页面文件，例如：

- `zh/search.md`：

```yaml
---

title: " 搜索 "
layout: "search"
summary: " 搜索页面 "
placeholder: " 这里可以输入..."
---
```

- `en/search.md`：

```yaml
---
title: "Search"
layout: "search"
summary: "Search page"
placeholder: "You can type here..."
---
```

然后对 `hugo.toml` 做最后的修改，设置默认显示语言，以及指定各语言所在目录。

```toml
baseURL                = "https://example.org/"
languageCode           = "zh"
defaultContentLanguage = "zh" # <<--- 新添加项
title                  = "ExampleSite"
theme                  = "PaperMod"

...

[languages]
  [languages.zh]
    languageCode = "zh"
    languageName = "中文"
    contentDir = "content/zh" # <<--- 新添加项
    weight       = 1

    [languages.zh.menu]
      ...
  [languages.en]
    languageCode = "en"
    languageName = "English"
    contentDir = "content/en" # <<--- 新添加项
    weight       = 2

    [languages.en.menu]
      ...
```

> **提示**：在 `defaultContentLanguage` 属性后面，可以继续添加 `defaultContentLanguageInSubdir = true` 开启默认语言内容生成在子目录中。

## 参考

- [Quick start | HUGO](https://gohugo.io/getting-started/quick-start/)
- [Configure menus | HUGO](https://gohugo.io/configuration/menus/)
- [Multilingual mode | HUGO](https://gohugo.io/content-management/multilingual/)
- [Installation | hugo-PaperMod Wiki](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation)
