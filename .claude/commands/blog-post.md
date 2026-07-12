写一篇关于「$ARGUMENTS」的博客文章。按以下步骤执行，每步完成后继续下一步，**不要在中间停下来询问**，直到第 4 步才停止等待审核。

## Step 0：封面去重检查

在选封面之前，先运行以下命令找出 wallpaper 目录里还没被用过的图片：

```bash
# 1. 生成已用图片的哈希列表（含 banner.jpg，避免和首页主题图重复）
md5 source/img/banner.jpg source/img/p*.jpg | sed 's/MD5 (\(.*\)) = \(.*\)/\2 \1/' > /tmp/covers.txt

# 2. 生成 wallpaper 目录的哈希列表
md5 ~/Pictures/wallpaper/*.jpg | sed 's/MD5 (\(.*\)) = \(.*\)/\2 \1/' > /tmp/wallpapers.txt

# 3. 输出还没被用过的 wallpaper 文件路径
awk 'NR==FNR{used[$1]=1;next} !used[$1]{print $2}' /tmp/covers.txt /tmp/wallpapers.txt
```

从输出的文件列表里选一张，复制到 `source/img/` 并命名为 `pXX.jpg`（XX 为当前最大编号 +1）。

复制完成后立即压缩，控制体积：

```bash
sips -s format jpeg -s formatOptions 80 -Z 1920 source/img/pXX.jpg
```

## Step 1：用 writing-plans skill 制定写作计划

调用 `superpowers:writing-plans` skill，为这篇文章制定结构计划：
- 确定文章类型（教程/工具介绍/深度解析）
- 列出主要章节和每节要覆盖的核心内容
- 确认读者定位和文章长度预估

**计划文件写到用户级计划目录 `~/.claude/plans/`（文件名格式 `YYYY-MM-DD-文章主题.md`），不要写进仓库**——那里本来就是存放计划的地方，写完也无需清理。

## Step 2：写作

按照计划写出完整文章，遵守以下约定：
- 文件保存到 `source/_posts/` 下，文件名用中文或英文均可
- frontmatter 必须包含 title、date、categories、tags、description 字段
- **`date`（以及后续任何修改时要加的 `updated`）字段必须先跑 `date "+%Y-%m-%d %H:%M:%S"` 拿真实系统时间再填，不要凭感觉编时间**——历史上多次把 `updated` 写成瞎猜的时间，跟实际相差好几个小时
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

## Step 4：起公网预览地址，停止等待审核

**启动前先检查有没有残留的旧进程**（比如我之前手动起过、忘了关）：
1. 检查端口 4000：`lsof -ti:4000`，有 PID 就 `kill` 掉，并告诉我"检测到旧的 hexo server 还在跑，已停止"
2. 检查 Tailscale funnel 进程：`pgrep -fl "tailscale funnel"`（**不要用** `tailscale funnel status`——前台模式的 funnel 不会同步进这个状态查询，会显示"No serve config"，查不到实际在跑的进程）。macOS 上 Tailscale.app 的 CLI 是个包装脚本，会连带跑出父子两个进程（`/bin/sh .../tailscale` 和 `.../Tailscale.app/.../tailscale`），**pgrep 匹配到几个 PID 就全部 kill 掉，只杀一个可能留下孤儿进程继续占着公网暴露**。杀完告诉我"检测到旧的 Tailscale funnel 还在跑，已停止"

**确认干净后再正常启动**：
1. 后台启动本地预览：`hexo clean && hexo server`
2. 用 Tailscale 前台模式代理到公网：`tailscale funnel 4000`（不加 `--bg`，配置跟这个进程绑在一起）
3. 从输出里取到公网访问地址——中间命令的过程和原始输出不用展示，只要这个地址

输出一段简短总结，**公网预览地址必须放在整段总结的最后面，不要提到最前面**（手机翻看时往上翻很不方便）：
- 文件路径
- 文章结构（各章节标题）
- 还有哪些可以扩展的方向（可选）
- 公网预览地址：**这是总结里最后一项，单独一行只放这个 URL**，后面不要紧跟括号说明、文章路径或其他文字——URL 和后面的内容连在一起会被识别成同一个（打不开的）链接。补充说明（比如"手机可直接打开"）放在 URL 前面单独一句，或者另起一行，不要接在 URL 后面

总结末尾问一句：审核完成后告诉我一声。

然后停下来，等我审核反馈。收到"停掉"指令时，直接终止 `hexo server` 和 `tailscale funnel` 这两个进程即可——前台模式下进程退出会自动清掉公网暴露配置，不用额外跑 `off` 命令。

我确认审核完成后：把这次要 commit 的内容（新文章 + 相关改动，比如同步更新过的地图页）整理成 commit message 发给我看，**先等我明确回复"OK"或确认无误，才执行 `git commit`**——不要自己生成完消息就直接 commit。

commit 完之后，我说"发布"/"部署"/"上线"/"推送"这类词，按 CLAUDE.md 里的部署命令执行，不用重新调用这个 skill。
