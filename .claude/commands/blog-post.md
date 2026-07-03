写一篇关于「$ARGUMENTS」的博客文章。按以下步骤执行，每步完成后继续下一步，**不要在中间停下来询问**，直到第 4 步才停止等待审核。

## Step 0：封面去重检查

在选封面之前，先运行以下命令找出 wallpaper 目录里还没被用过的图片：

```bash
# 1. 生成已用封面的哈希列表
md5 source/img/p*.jpg | sed 's/MD5 (\(.*\)) = \(.*\)/\2 \1/' > /tmp/covers.txt

# 2. 生成 wallpaper 目录的哈希列表
md5 ~/Pictures/wallpaper/*.jpg | sed 's/MD5 (\(.*\)) = \(.*\)/\2 \1/' > /tmp/wallpapers.txt

# 3. 输出还没被用过的 wallpaper 文件路径
awk 'NR==FNR{used[$1]=1;next} !used[$1]{print $2}' /tmp/covers.txt /tmp/wallpapers.txt
```

从输出的文件列表里选一张，复制到 `source/img/` 并命名为 `pXX.jpg`（XX 为当前最大编号 +1）。

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

## Step 3：用 technical-blog-writing、blog-writing-guide、seo-content-writer 和 content-quality-auditor skill 优化

依次调用：
1. `technical-blog-writing` skill：对照技术博客写作标准整体检查，包括代码示例是否可运行、技术表述是否准确
2. `blog-writing-guide` skill：检查开头是否直接切入、标题是否传递信息、有无 AI 写作套话（平行结构广告体、格言体等）
3. `seo-content-writer` skill：检查 description 字段、H2 关键词覆盖
4. `content-quality-auditor` skill：综合内容质量把关，检查结构完整性、可读性、事实声明是否有据可查

发现问题直接修改，不要列出问题清单再问我要不要改。

## Step 3.5：准确性与时效性核查

对文章中所有具体的技术声明进行核查，确保内容准确、不过时：

**需要核查的内容**（逐条检查，有疑问就用 WebSearch 或 defuddle skill 查官方文档）：
- 版本号：文中提到的工具/库版本是否仍是当前版本或主流版本
- 命令和参数：命令语法是否和官方文档一致，参数是否仍然有效
- API / 配置字段：接口和配置项是否已变更或废弃
- 安装方式：brew、npm、curl 等安装命令是否仍然适用
- 外部链接：引用的官方文档地址是否有效

**处理原则**：
- 核实无误 → 不改动
- 发现过时或有误 → 直接修正，在 Step 4 的总结里注明改了什么
- 无法核实（无网络或文档找不到）→ 在内容里加注"建议以官方文档为准"或标注版本适用范围

## Step 4：停止，等待审核

输出一段简短总结：
- 文件路径
- 文章结构（各章节标题）
- 还有哪些可以扩展的方向（可选）

然后停下来，等我审核反馈。
