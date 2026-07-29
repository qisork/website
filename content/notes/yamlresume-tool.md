---
title: 寻找制作简历的工具（YAMLResume）
date: 2025-10-06T21:14:22+08:00
tags:
  - yamlresume
  - latex
  - 评测
draft: false
description: 本文介绍了YAMLResume工具的使用方法，包括通过Docker命令创建简历模板和生成PDF文件，以及该工具的优缺点分析。
summary: 本文介绍了YAMLResume工具的使用方法，包括通过Docker命令创建简历模板和生成PDF文件，以及该工具的优缺点分析。
---

正如提交的答卷一般，易于阅读的排版，恰当的用词，总能加不少印象分。如果我的简历能让人一目了然，短时间内了解我，那我的简历就是成功的，至于会不会录取这个另说。

同时为了加快简历的制作，不必囚禁在排版/格式调整的死循环中，从最开始的 Word 模板、画布模板中开始尝试，直到我接触到了 LaTeX 简历模板，直呼一声“哇”，这就是我想要的。

跟 markdown 一样，只需使用标记符标记好内容，就完全不需要考虑排版问题，不会像 Word 那样，布局上下乱跳，只需要关心内容就好。

鉴于我并不会 LaTeX 语法，同时也想快速制作简历，所以接下来我找到了 [YAMLResume](https://yamlresume.dev/)，更为易用的工具。通过使用 yaml 配置语法，简化 LaTeX 的编写，但实际是将 yaml 内容转换为 LaTeX，然后编译 LaTeX 文件。

![YAMLResume 的效果图](https://img.qisrepo.top/img-yamlresume-yaml-and-pdf.webp)

安装也特别的方便，提供了 docker 镜像，只需要拉取运行，无需担心环境问题，就是空间占用比较大，3G 左右。

- `new`：运行此命令会创建一个[示例模板](https://yamlresume.dev/docs#sample-resume)

```bash
docker run --rm -v $(pwd):/home/yamlresume yamlresume/yamlresume new my-resume.yml
```

- `build`：将 new 生成的文件，导出为 PDF 格式的简历

```bash
docker run --rm -v $(pwd):/home/yamlresume yamlresume/yamlresume build my-resume.yml
```

> 如果想使用本地安装的方式，请参考[官方文档](https://yamlresume.dev/docs/installation#non-docker-users)。

使用 `new` 命令创建一个示例模板，然后根据提供的示例编写对应内容，然后使用 `build` 命令生成 PDF 格式的简历。

YAMLResume 还提供了一些自定义的设置，比如：

- 为章节标题设置[别名](https://yamlresume.dev/docs/layout/sections/aliases)
- 对简历章节的显示顺序进行[自定义排序](https://yamlresume.dev/docs/layout/sections/reorder)

到这里，已经能够编写出一份不错的简历了，开始总结。

YAMLResume 的优点在于易于使用，同时也有其缺陷：

- 对页面的自定义程度很弱，如果想为章节标题增加一些图标，必须改动主题模板。
- `projects` 章节下的 `description` 不能自动换行，一旦文本过长，就会超出页面显示范围。

---

- [YAMLResume](https://yamlresume.dev/)
- [Types - YAMLResume](https://yamlresume.dev/docs/compiler/types)
- [Rich Text - YAMLResume](https://yamlresume.dev/docs/content/rich-text)

