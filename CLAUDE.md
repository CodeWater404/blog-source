# Blog Project — Claude 约定

## 项目概览

- 博客框架：Hexo 7.3.0
- 托管：GitHub Pages，自定义域名
- 文章格式：Markdown
- 文章目录：`source/_posts/`

## Frontmatter 模板

每篇文章必须包含以下字段：

```yaml
---
title: "文章标题"
date: YYYY-MM-DD HH:MM:SS
categories: 分类名   # 单个分类，如 golang / tools / tutorial
tags:
  - 标签1
  - 标签2
description: "一句话摘要，用于 SEO meta description 和社交分享卡片，控制在 80 字以内。"
---
```

## 文件命名规则

- 直接使用中文或英文均可，和文章主题相关
- 示例：`Mac终端基础-kitty和Zim.md`、`Go后端实现Google一键登录.md`
- 不用加日期前缀，Hexo 从 frontmatter 的 date 字段读取

## 系列文章约定

当一篇文章有对应的详细版时：

1. **汇总篇**：在对应章节标题下方加 blockquote 引用，格式：
   ```markdown
   > 详细安装与配置见：[文章标题](/文件名不带扩展名)
   ```
2. **详细篇**：章节标题直接带官网链接，格式：
   ```markdown
   ## [工具名](https://官网链接)：副标题
   ```

## 内部链接格式

Hexo 内部链接用 `/文件名`（不带 `.md` 扩展名），例如：

```markdown
[Mac 终端 CLI 工具推荐](/Mac终端CLI工具推荐)
```

## 部署命令

```bash
hexo clean && hexo g -d
```

## 写作规范

- 文章写到 `source/_posts/` 下，不使用 `_drafts`
- 每篇文章必须有 `<!-- more -->` 分隔符，控制首页摘要
- 不在文章里写"本文将介绍……"类的开场声明，直接进入内容
- 章节标题要传递信息，不用"简介""总结""背景"这类空洞标题
- 命令教程类文章，代码块里出现的参数必须用注释说明含义，格式：

  ```bash
  git tag -a v1.0.0 -m "Release v1.0.0"
  #      -a：annotated，创建附注标签
  #      -m：message，标签说明，省略会弹出编辑器
  ```
