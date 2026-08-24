# AGENTS.md

本文件为跨 AI 工具通用的项目规则。任何 AI（Claude、ChatGPT、Gemini、Cursor、Codex、DeepSeek 等）在为本项目生成 Git 提交信息时，必须严格遵守以下规范。

## 项目性质

本仓库是一个 **Hugo 内容网站**，而非传统软件开发项目。生成提交信息时，**不要**套用学术界/工程界主流的 Conventional Commits 类型（如 `feat`、`refactor`、`perf`、`test`、`ci` 等），而是使用下方为内容项目定制的类型体系。

## 提交信息结构

每条提交信息由两部分组成：**Header**（必填）、**Body**（选填）。

```
<type>(<scope>): <subject>	// Header

<body>						// Body
```

约束：
- Header 与 Body 之间用空行分隔
- Header 行不超过 **72** 个字符
- Body 每行不超过 **80** 个字符

## Type 类型（唯一允许的枚举）

生成提交信息时，`type` 只能从以下列表中选择：

| 类型 | 说明 | 示例场景 |
| ---- | ---- | ---- |
| `post` | 新增/更新文章 | 发布新博客、修改已有文章 |
| `content` | 非文章的页面内容变更 | 修改关于页、友链页、归档页等独立页面 |
| `theme` | 主题相关改动 | 调整模板、布局、样式 |
| `config` | 配置改动 | 修改 `hugo.toml`、菜单、站点参数等 |
| `asset` | 资源改动 | 增删图片、字体、CSS、JS 等静态文件 |
| `fix` | 修复问题 | 页面渲染错误、链接失效、排版问题 |
| `chore` | 构建/工具链变更 | CI/CD、`Makefile`、部署脚本 |
| `docs` | 文档变更 | README、贡献指南等非网站内容 |
| `revert` | 回滚提交 | 撤销先前的某次提交 |

禁止使用上述以外的任何类型（尤其是 `feat`、`refactor`、`perf`、`test`、`build`、`ci`、`style` 等）。

## Scope 范围

`scope` 优先使用 Hugo 项目中的目录名，用于快速定位改动位置：

```
posts/          博客文章
pages/          独立页面
layouts/        模板文件
static/         静态资源
assets/         资源文件（SCSS/JS 等）
data/           数据文件
i18n/           国际化翻译
config/         配置目录
archetypes/     内容模板
themes/         主题文件
```

补充规则：
- 文章类提交使用文章的 **slug** 作为 scope，例如 `container-deploy`、`edge-computing`
- 当改动只涉及单个文件时，可直接用 **文件名** 作为 scope，例如 `about.md`、`main.css`
- 无法确定 scope 时可省略，但应先判断改动归属

## Subject 描述

- 以动词开头，使用祈使语气，末尾**不加句号**
- 长度不超过 **50** 个字符
- 优先使用中文，同一项目内保持一致

正确示例：

```
post(change-hugo-theme): 新增关于更换 Hugo 主题的文章
fix(posts/): 修复某某文章中的错别字
config(hugo.toml): 启用代码高亮并设置行号显示
theme(theme-name): 升级主题至 v2.1.0
```

错误示例（必须避免）：

```
修改了一些东西                      # 缺少类型，描述模糊
post: 改文章                       # 主题行过于简略
fix(layouts): 修了个bug。          # 末尾多了句号
```

## Body 详细说明

正文用于说明变更动机与上下文，非必需，但在改动较大时应补充。要求：
- 每行不超过 80 个字符
- 说明「为什么」而非复述「改了什么」

示例：

```
post(change-hugo-theme): 新增关于更换 Hugo 主题的文章

文章涵盖主题选择思路、从某主题迁移到某主题的完整过程、
旧文章 front matter 字段适配，以及自定义配色方案。
对比截图已添加至 content/posts/change-hugo-theme/ 目录。
```

## 特殊约定

- **草稿文章**：未发布的草稿在主题行中加 `[draft]` 前缀
  ```
  post(change-hugo-theme): [draft] 新增关于更换 Hugo 主题的文章
  ```

- **配图资源**：文章配图建议与正文分开提交，或在主题行中明确标注
  
  ```
  post(change-hugo-theme): 添加文章配图
  ```
  
- **主题子模块**：主题以 Git submodule 管理时，更新采用固定格式
  ```
  theme(theme-name): 更新 theme-name 主题 submodule 至 commit <hash>
  ```

## 提交频率参考

生成提交信息时，如无法从改动中自动判断粒度，可参考以下拆分原则：

| 场景 | 建议 |
| ---- | ---- |
| 写一篇文章 | 文章内容一个提交，配图可合并或单独提交 |
| 批量修正错别字或排版 | 合并为一个 `fix` 提交 |
| 主题大改 | 按模板拆分，每个模板一个提交 |
| 配置文件调整 | 一次调整对应一个提交 |

## 生成提交信息时的行为要求

1. 先分析实际改动内容（文件路径、改动类型、是否涉及多文件），再映射到正确的 `type` 与 `scope`
2. 禁止在未判断改动性质的情况下，一律使用 `post` 或其他单个固定类型
3. 保持中英文混用规则一致：`type`/`scope` 使用英文，`subject` 描述优先中文
4. 输出结果必须可直接用作 `git commit -m` 的内容，不要附加多余解释（除非用户要求说明）