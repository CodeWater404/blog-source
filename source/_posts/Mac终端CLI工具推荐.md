---
title: "Mac 终端 CLI 工具推荐：那些让命令行效率翻倍的家伙"
date: 2026-06-28 10:00:00
categories: tools
tags:
  - Mac
  - CLI
  - Terminal
  - 效率工具
description: "系统梳理 Mac 终端完整工具链：从终端模拟器 kitty、Shell 框架 Zim，到 fzf、ripgrep、lazygit、LazyVim 等核心工具，聚焦每个工具的核心价值。"
---

macOS 自带的终端工具集（`grep`、`find`、`ls`、`cat`……）能用，但有些地方用起来总觉得别扭——搜索慢、输出不直观、没有高亮、参数难记。过去几年，社区里涌现出一批用 Rust、Go 写的现代 CLI 工具，直接解决了这些痛点。

<!-- more -->

---

## 终端模拟器：[kitty](https://sw.kovidgoyal.net/kitty/)

> 详细安装与配置见：[Mac 终端基础配置：kitty + Zim](/Mac终端基础-kitty和Zim)

所有工具的运行基础。kitty 是一个用 GPU 渲染的高性能终端模拟器，启动快、滚动流畅，原生支持图片内联显示（这也是 yazi 文件预览图片能用的前提）。内置分屏（不依赖 tmux），支持标签页，配置文件是单个 `kitty.conf`，改起来比大多数终端直观。对于在终端里花大量时间的人，换一个渲染性能好的终端模拟器是性价比很高的基础投资。

---

## Shell 框架：[Zim](https://zimfw.sh/)

> 详细安装与配置见：[Mac 终端基础配置：kitty + Zim](/Mac终端基础-kitty和Zim)

macOS 自带 zsh，但裸 zsh 体验很差。Zim 是一个轻量的 zsh 插件管理框架，比 Oh My Zsh 启动速度快很多，通过 `.zimrc` 声明需要的插件，装好后自动生效。

当前配置里启用的插件：

- **fast-syntax-highlighting**：实时高亮正在输入的命令，命令合法显绿色，输错显红色，按回车前就能发现拼写错误
- **zsh-autosuggestions**：根据历史记录实时预测补全建议，显示为灰色文字，按 `→` 接受，减少大量重复输入
- **zsh-history-substring-search**：按 `↑↓` 方向键时，按当前已输入内容做前缀匹配过滤历史，而不是顺序翻
- **fzf-tab**：把 Tab 补全的候选列表接管给 fzf，变成可交互过滤的模糊搜索界面
- **steeef**：Prompt 主题，展示用户名、当前路径、Git 分支和状态

---

## Shell 增强工具

### fzf

模糊搜索框架，单独使用时是一个交互式过滤器，真正的价值在于**和其他命令组合**。集成进 Zsh 后会接管三个快捷键：`Ctrl+T` 模糊搜索文件、`Alt+C` 模糊跳转目录、`Ctrl+R` 交互式搜索命令历史。配合上面的 fzf-tab，Tab 补全也变成模糊搜索界面。fzf 是整个工具链里"乘数效应"最强的一个——它本身不做什么，但让其他工具都变得更好用。

### atuin

shell 历史记录的替代方案，把命令历史存进本地 SQLite 数据库，支持按时间、工作目录、退出码等维度检索。最直接的改变是 `Ctrl+R` 变成了带预览的全文搜索界面。还支持多设备历史同步，在不同 Mac 之间共享命令历史。

### zoxide

`cd` 的智能替代品。记住访问过的目录，之后输入 `z blog` 就能直接跳到 `/Users/you/code/github/blog-source`，不需要输完整路径。项目目录多、路径深的开发者体验提升非常明显，是少数几个"装了就再也不想卸"的工具之一。

---

## 终端个性化

### fastfetch + pokemon-colorscripts

这两个组合使用：每次打开新终端，先随机展示一只宝可梦的彩色 ASCII 图，fastfetch 把系统信息（OS、CPU、内存、Shell、终端等）渲染在旁边。纯个性化，无实际功能，但比默认的空终端有意思很多。fastfetch 用 C 写成，启动速度远快于老牌的 neofetch，不会拖慢终端启动。

---

## 文件与搜索

### ripgrep（rg）

比 `grep` 快一个数量级的全文搜索工具，默认递归搜索当前目录，自动忽略 `.gitignore` 里的文件。在大型代码仓库里搜索关键词，`rg` 的速度优势非常明显，输出带文件名、行号和彩色高亮。VSCode 的全局搜索底层用的也是 ripgrep。

### fd

`find` 的现代替代品。默认大小写不敏感、忽略 `.gitignore` 内容、输出带颜色，语法比 `find` 直观很多——找 Go 文件直接 `fd -e go`，而不是 `find . -name "*.go" -type f`。和 fzf 结合使用效果更好。

### bat

`cat` 的替代品，带语法高亮、行号，还有 Git 集成（在行号旁标注哪些行被修改过）。它也被很多其他工具用作底层渲染器——fzf 的文件预览、yazi 的代码预览默认都会调用 bat。

### eza

`ls` 的现代替代品。默认输出带颜色，`-l` 模式下可以显示 Git 状态，`--tree` 模式展示目录树——这个功能直接取代了单独装 `tree` 命令的需求。配合 Nerd Font 字体还能显示文件类型图标，比原生 `ls -la` 的信息密度高很多。

### yazi

终端里的文件管理器，基于 Rust 构建，速度很快。方向键导航目录、实时预览文件内容（代码、图片、PDF），批量操作文件。在 kitty 终端下能直接内联显示图片预览。对于不想离开终端但又需要浏览文件的场景，yazi 比反复 `cd` + `ls` 高效很多。`.zshrc` 里定义了 `y` 函数，退出 yazi 时会自动 `cd` 到最后所在目录。

### jq

JSON 的命令行处理器，核心能力是**对 JSON 进行查询、过滤、转换**。调接口拿到 JSON 响应后，`curl ... | jq '.data.items[] | .name'` 这样的管道就能直接提取想要的字段。处理 API 响应、配置文件、日志分析时几乎不可缺少。

---

## Git 工具

### git-delta

Git diff 的语法高亮渲染器。配置进 `~/.gitconfig` 后，`git diff`、`git log -p`、`git show` 的输出变成带语法高亮、行内差异标注的格式，比默认的加减号 diff 可读性高出不少。

### lazygit

终端里的 Git TUI，用键盘操作就能完成暂存、提交、分支管理、rebase、stash 等常用 Git 操作，不用记忆大量命令参数。`.zshrc` 里设置了 `alias lg=lazygit`，快速调出。关于 lazygit 的常用工作流，后续会单独介绍。

### gh

GitHub 官方 CLI。核心价值在于**不离开终端就能操作 GitHub**——创建 PR、查看 Issue、管理 Actions 工作流、查看 Review 评论。过去这些操作要切到浏览器，现在在命令行里完成，减少了大量上下文切换，在 CI/CD 调试、PR Review 来回改代码这类场景下体感尤其明显。

---

## Markdown 渲染：glow / mdcat

### glow

终端里的 Markdown 渲染器。`glow README.md` 直接把 Markdown 渲染成格式化输出——标题、加粗、代码块都有对应的视觉呈现。看项目文档、博客草稿时比 `cat` 体验好很多。

### mdcat

同样是 Markdown 渲染工具，更轻量，在 kitty 终端下支持内联显示图片。文档里有图的场景，mdcat 的图文混排体验比 glow 更完整。

---

## 编辑器：Neovim + LazyVim

Neovim 是 Vim 的现代分支，支持 Lua 配置、LSP、Tree-sitter 语法高亮。但从零配置 Neovim 到可用状态需要花大量时间。

**LazyVim** 解决了这个问题——它是一套预配置的 Neovim 发行版，开箱即得代码补全、LSP、文件树、模糊搜索、Git 集成等功能。底层插件管理器是 **lazy.nvim**，按需懒加载插件，启动速度快。

配套工具 **im-select** 负责输入法自动切换：从 Neovim 的 Insert 模式退出时，自动把输入法切换回英文，避免进入 Normal 模式后还在中文状态下输入命令的问题。

LazyVim 的上手曲线比从零配置 Neovim 低很多，但整体还是需要适应 Vim 的模式操作。关于 LazyVim 的配置和常用插件，后续会单独介绍。

---

## 从哪里开始

按学习成本从低到高：

- **零成本，立竿见影**：ripgrep、bat、eza、git-delta、zoxide、atuin——装上改个别名就能用
- **需要适应键位**：fzf、lazygit、yazi——花半小时熟悉一下，之后回报持续
- **需要系统投入**：LazyVim——上手慢，但配置完善后是全功能命令行 IDE

后续会针对 fzf、lazygit、LazyVim 单独写深入使用的文章。
