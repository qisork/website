---
title: 用于 Hugo 网站的 Git 提交规范
slug: hugo-site-git-commit-convention
date: 2026-08-24T20:44:36+08:00
categories: 
  - 技术
tags:
  - git
draft: false
description: "为 Hugo 网站定制的一套 Git 提交规范，涵盖类型、范围、主题描述及 commitlint 自动化校验配置。"
---

随着，我对网站的修改量增加，发现 Git 提交信息异常混乱，包括不限于修改类型不一致、范围不正确等等。

这主要是我为了偷懒，采用了 AI 自动生成 Git 提交信息，很显然 AI 并没有因地制宜的处理，而是直接照搬主流的 Git 提交规范，忽略了我的网站不是一个开发项目，而是一个内容项目。

为了解决这个问题，我用 AI 专门为我的 Hugo 网站生成了一套 Git 提交规范。

## Git 提交规范

### 提交消息结构

每条提交消息由两部分组成：**Header**（必填）、**Body**（选填）。

```text
<type>(<scope>): <subject>	// Header

<body>						// Body
```

- 各部分之间用空行分隔
- Header 行不超过 **72** 个字符
- Body 每行不超过 **80** 个字符

### Type 类型

根据 Hugo 网站的特点，定义以下类型：

| 类型      | 说明                 | 示例场景                             |
| --------- | -------------------- | ------------------------------------ |
| `post`    | 新增/更新文章        | 发布新博客、修改已有文章             |
| `content` | 非文章的页面内容变更 | 修改关于页、友链页、归档页等独立页面 |
| `theme`   | 主题相关改动         | 调整模板、布局、样式                 |
| `config`  | 配置改动             | 修改 `hugo.toml`、菜单、站点参数等   |
| `asset`   | 资源改动             | 增删图片、字体、CSS、JS 等静态文件   |
| `fix`     | 修复问题             | 页面渲染错误、链接失效、排版问题     |
| `chore`   | 构建/工具链变更      | CI/CD、`Makefile`、部署脚本          |
| `docs`    | 文档变更             | README、贡献指南等非网站内容         |
| `revert`  | 回滚提交             | 撤销先前的某次提交                   |

### Scope 范围

scope 优先使用 Hugo 项目中的目录名，帮助快速定位改动位置：

```text
posts/          博客文章
pages/          独立页面
layouts/        模板文件
static/         静态资源
assets/         资源文件（SCSS/JS等）
data/           数据文件
i18n/           国际化翻译
config/         配置目录
archetypes/     内容模板
themes/         主题文件
```

- 文章类提交使用文章的 **slug** 作为 scope，例如 `container-deploy`、`edge-computing`
- 当改动只涉及单个文件时，可直接用**文件名**作为 scope，例如 `about.md`、`main.css`
- 无法确定 scope 时可省略，但应先判断改动归属

### Subject 描述

- 以动词开头，使用祈使语气，末尾不加句号
- 长度不超过 50 个字符
- 优先使用中文，同一项目内保持一致

```text
# 推荐
post(change-hugo-theme): 新增关于更换 Hugo 主题的文章
fix(posts/): 修复某某文章中的错别字
config(hugo.toml): 启用代码高亮并设置行号显示
theme(theme-name): 升级主题至 v2.1.0

# 不推荐
修改了一些东西                     缺少类型，描述模糊
post: 改文章                      主题行过于简略
fix(layouts): 修了个bug。         末尾多了句号
```

### Body 详细说明

正文用于说明变更动机和上下文，不是必须的，但在改动较大时建议写上：

```text
post(change-hugo-theme): 新增关于更换 Hugo 主题的文章

文章涵盖主题选择思路、从某主题迁移到某主题的完整过程、
旧文章 front matter 字段适配，以及自定义配色方案。
对比截图已添加至 content/posts/change-hugo-theme/ 目录。
```

## 建议

### 草稿文章

未发布的草稿在主题行中加 `[draft]` 前缀，方便在提交历史中一眼识别：

```text
post(change-hugo-theme): [draft] 新增关于更换 Hugo 主题的文章
```

### 配图资源

文章配图建议与正文分开提交，或在主题行中明确标注：

```text
post(change-hugo-theme): 添加文章配图
```

### 主题子模块

主题以 Git submodule 形式管理时，更新使用固定格式：

```text
theme(theme-name): 更新 theme-name 主题 submodule 至 commit <hash>
```

### 提交频率建议

| 场景                 | 建议                                   |
| -------------------- | -------------------------------------- |
| 写一篇文章           | 文章内容一个提交，配图可合并或单独提交 |
| 批量修正错别字或排版 | 合并为一个 `fix` 提交                  |
| 主题大改             | 按模板拆分，每个模板一个提交           |
| 配置文件调整         | 一次调整对应一个提交                   |

### Commitlint 配置参考

可在项目根目录添加 `.commitlintrc.yml` 进行自动化校验：

```yaml
extends:
  - '@commitlint/config-conventional'
rules:
  type-enum:
    - 2
    - always
    - [post, content, theme, config, asset, fix, chore, docs, revert]
  subject-max-length:
    - 2
    - always
    - 50
  header-max-length:
    - 2
    - always
    - 72
```

配合 `husky` 安装 Git hook 即可在提交时自动校验：

```shell
npm install -D @commitlint/cli @commitlint/config-conventional husky
npx husky install
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit ${1}'
```