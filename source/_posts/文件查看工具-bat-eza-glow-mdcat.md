---
title: "文件查看升级：bat + eza + glow + mdcat"
date: 2026-07-01 10:00:00
categories: tools
tags:
  - Mac
  - CLI
  - Terminal
  - 效率工具
description: "Mac 终端文件查看升级：bat 替换 cat、eza 替换 ls、glow 和 mdcat 渲染 Markdown，含安装、alias 配置和常用参数速查。"
---

`cat`、`ls`、`less` 这些命令能用，但输出是黑白平铺的文字，没有高亮、没有结构感。bat、eza、glow、mdcat 分别解决了这四个场景的可读性问题，装完改几个 alias，原来的使用习惯不变，输出变好看很多。

<!-- more -->

---

## [bat](https://github.com/sharkdp/bat)：cat 加上语法高亮和 Git 标注

bat 是 `cat` 的替代品，核心差别：

- 自动语法高亮（支持上百种语言，通过 syntect 库驱动）
- 显示行号
- Git 集成：在行号旁标注哪些行被修改、新增、删除
- 自动分页（内容超过一屏时调用 pager，不会刷屏）

它也被很多其他工具用作渲染底层——fzf 的文件预览、yazi 的代码预览默认都会调用 bat。

### 安装

```bash
brew install bat
```

### 常用参数

| 参数 | 说明 |
|------|------|
| `-n` / `--number` | 强制显示行号（默认已开启） |
| `-A` / `--show-all` | 显示不可见字符（tab、空格、换行符） |
| `--plain` / `-p` | 纯输出模式，去掉行号和边框，适合管道传递 |
| `--language <lang>` | 手动指定语言（如 `--language json`） |
| `--theme <name>` | 指定高亮主题 |
| `--list-themes` | 列出所有可用主题 |
| `--pager never` | 禁用分页，直接输出全部内容 |
| `-r <start>:<end>` | 只显示指定行范围（如 `-r 10:20`） |

```bash
# 查看文件，自动高亮
bat main.go

# 预览主题效果
bat --list-themes | fzf --preview="bat --theme={} --color=always main.go"
#                  ----|fzf 预览窗口实时切换主题效果

# 只输出第 10-20 行
bat -r 10:20 main.go
#   -r：range，指定行范围

# 纯内容输出（管道给其他命令用）
bat -p config.json | jq '.database'
#   -p：plain，去掉装饰，等同于 cat
```

### 替换 cat：alias 设置

在 `~/.zshrc` 里加：

```bash
alias cat="bat"
alias catp="bat --plain"  # 需要纯输出时用 catp
```

### 主题选择

bat 内置了多个主题，`Dracula`、`Nord`、`Solarized (dark)` 比较常见。把选好的主题写进配置文件，不用每次手动指定：

```bash
# 查看当前配置文件位置
bat --config-file

# 编辑配置文件，加入主题设置
echo '--theme="Dracula"' >> "$(bat --config-file)"
```

---

## [eza](https://eza.rocks/)：ls 的现代替代品

eza 是 `ls` 的替代品（从已停止维护的 `exa` fork 而来），默认输出带颜色，额外功能：

- `-l` 模式显示 Git 状态（文件有没有被修改、是否未追踪）
- `--tree` 模式展示目录树，取代单独装 `tree` 命令
- 配合 Nerd Font 字体可以显示文件类型图标

### 安装

```bash
brew install eza
```

### 核心用法

```bash
eza                    # 等同于 ls，彩色输出
eza -l                 # 长格式，显示权限、大小、修改时间
eza -la                # 包含隐藏文件
eza --tree             # 树形展示当前目录
eza --tree -L 2        # 树形展示，深度限制为 2 层
eza -l --git           # 长格式 + Git 状态列
```

### 常用参数

| 参数 | 说明 |
|------|------|
| `-l` / `--long` | 长格式，显示详细信息 |
| `-a` / `--all` | 包含隐藏文件（`.` 开头） |
| `-T` / `--tree` | 树形展示目录结构 |
| `-L <n>` / `--level <n>` | 树形展示的深度限制 |
| `--git` | 长格式中增加 Git 状态列 |
| `--icons` | 显示文件类型图标（需要 Nerd Font） |
| `-s <field>` / `--sort <field>` | 排序字段，如 `size`、`modified`、`ext` |
| `-r` / `--reverse` | 逆序排列 |
| `--no-permissions` | 不显示权限列（输出更简洁） |
| `-h` / `--header` | 长格式时显示列标题 |

```bash
# 按文件大小倒序排列
eza -l -s size -r
#      -s：sort，指定排序字段
#             -r：reverse，逆序

# 带图标的树形展示，深度 3 层
eza --tree --icons=always -L 3
#          --icons=always：显示图标（需要 Nerd Font）
#                          -L：level，树的深度

# 查看目录，不递归进子目录
eza -lD
#   -D：只显示目录
```

### alias 配置

在 `~/.zshrc` 里加：

```bash
alias ls="eza"
alias ll="eza -l --git"
alias la="eza -la --git"
alias lt="eza --tree -L 2"
alias llt="eza -l --tree -L 2 --git"
```

---

## [glow](https://github.com/charmbracelet/glow)：终端里渲染 Markdown

glow 把 Markdown 渲染成格式化的终端输出，标题有缩进和颜色，加粗正常显示，代码块有高亮，比直接 `cat` 看 `.md` 文件体验好很多。

### 安装

```bash
brew install glow
```

### 常用参数

| 参数 | 说明 |
|------|------|
| `-p` / `--pager` | 用内置分页器打开（支持上下滚动和搜索） |
| `-w <n>` / `--width <n>` | 指定渲染宽度（默认跟随终端宽度） |
| `-s <style>` / `--style <style>` | 主题，可选 `dark`、`light`、`notty`、`dracula` |

```bash
# 渲染单个文件
glow README.md

# 用分页器打开（可以上下滚动）
glow -p README.md
#    -p：pager，开启分页模式

# 指定换行宽度，适合宽屏
glow -w 100 README.md
#    -w：width，自动换行的列数（设为 0 禁用自动换行）

# 交互式浏览当前目录所有 .md 文件
glow
```

不加参数直接运行 `glow`，会列出当前目录里的所有 Markdown 文件供选择，适合快速翻看项目文档。

---

## [mdcat](https://github.com/swsnr/mdcat)：图文混排的 Markdown 渲染

mdcat 和 glow 功能类似，核心区别：**在 kitty 终端下支持内联显示图片**。Markdown 文档里如果有图片，mdcat 会直接把图片渲染在终端里，而不是显示一个链接。

### 安装

```bash
brew install mdcat
```

### 用法

```bash
# 渲染 Markdown 文件（默认不分页，直接输出）
mdcat README.md

# 开启分页（内容多时可上下滚动）
mdcat --paginate README.md
#     --paginate / -p：启用分页器（类似 less），默认是关闭的
```

### glow vs mdcat 怎么选

| | glow | mdcat |
|--|------|-------|
| 图片内联显示 | 不支持 | 支持（需要 kitty） |
| 交互式浏览 | 支持（直接运行 `glow`） | 不支持 |
| 主题自定义 | 更多选项 | 较少 |
| 适合场景 | 日常看文档、README | 文档里有图的场景 |

两个都装，看纯文字文档用 glow，文档里有图用 mdcat。

---

## bat 作为底层渲染器

安装好 bat 之后，其他工具会自动调用它，不需要额外配置：

- **fzf 文件预览**：`Ctrl+T` 预览文件时，bat 负责渲染高亮
- **yazi 代码预览**：打开代码文件时，yazi 调用 bat 渲染
- **git-delta**：diff 输出的语法高亮底层也依赖 bat 的主题

bat 的主题配置会统一影响这些工具的显示效果，选主题时值得多试几个。

---

## 常见问题

**bat 和 cat 有什么区别，能直接替换吗？**

bat 是 cat 的超集，所有 cat 能做的 bat 都能做，额外增加语法高亮、行号、Git 标注和自动分页。用 `alias cat="bat"` 直接替换没有问题。如果某些脚本需要纯文本输出（不带颜色控制符），用 `bat --plain` 或单独保留 `catp` 别名。

**eza 和 exa 什么关系？**

eza 是从已停止维护的 exa fork 出来的社区维护版本，功能上基本兼容，额外修了一些 bug 并持续更新。如果你之前用 exa，换成 eza 直接改 alias 就行，命令参数基本相同。

**glow 和 mdcat 图片显示的区别是什么？**

glow 不支持内联图片，Markdown 里的图片会显示为纯文字链接。mdcat 在 kitty 终端下支持把图片直接渲染在终端里（Sixel 协议或 kitty 图形协议），文档里有配图的场景体验更完整。如果不用 kitty，两者在图片处理上没有差别。

**这四个工具需要 Nerd Font 吗？**

bat、glow、mdcat 不需要。eza 的 `--icons` 参数需要安装 Nerd Font，否则图标会显示为乱码。不想装字体的话，不加 `--icons` 参数正常使用即可。
