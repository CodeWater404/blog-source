写一篇关于「$ARGUMENTS」的博客文章。按以下步骤执行，每步完成后继续下一步，**不要在中间停下来询问**，直到第 4 步才停止等待审核。

## Step 1：用 writing-plans skill 制定写作计划

调用 `superpowers:writing-plans` skill，为这篇文章制定结构计划：
- 确定文章类型（教程/工具介绍/深度解析）
- 列出主要章节和每节要覆盖的核心内容
- 确认读者定位和文章长度预估

## Step 2：写作

按照计划写出完整文章，遵守以下约定：
- 文件保存到 `source/_posts/` 下，文件名用中文或英文均可
- frontmatter 必须包含 title、date、categories、tags、description 字段
- 加上 `<!-- more -->` 分隔摘要
- 开头直接进入内容，不写"本文将介绍……"类的声明
- 章节标题要传递信息（不用"简介""背景""总结"这类空标题）
- 工具类文章的章节标题带上该工具的官网链接

## Step 3：用 technical-blog-writing、blog-writing-guide 和 seo-content-writer skill 优化

依次调用：
1. `technical-blog-writing` skill：对照技术博客写作标准整体检查，包括代码示例是否可运行、技术表述是否准确
2. `blog-writing-guide` skill：检查开头是否直接切入、标题是否传递信息、有无 AI 写作套话（平行结构广告体、格言体等）
3. `seo-content-writer` skill：检查 description 字段、H2 关键词覆盖

发现问题直接修改，不要列出问题清单再问我要不要改。

## Step 4：停止，等待审核

输出一段简短总结：
- 文件路径
- 文章结构（各章节标题）
- 还有哪些可以扩展的方向（可选）

然后停下来，等我审核反馈。
