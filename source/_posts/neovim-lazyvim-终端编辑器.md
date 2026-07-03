---
title: "从零开始用 Neovim：LazyVim 让配置不再是障碍"
date: 2026-07-03 18:00:00
cover: /img/p21.jpg
categories: tools
tags:
  - Mac
  - CLI
  - Terminal
  - Neovim
  - LazyVim
  - 效率工具
description: "LazyVim 是 Neovim 的完整配置框架，开箱即有 LSP、文件树、模糊搜索、Git 集成。本文从安装到基础操作，帮你跳过配置地狱直接进入使用。"
---

Neovim 最大的门槛不是学习 Vim 操作，而是配置。从零搭一套可用的 Neovim 环境，光是理解插件管理器、LSP 配置、补全框架的关系就要花很多时间。LazyVim 解决的正是这个问题：它是一套预先打包好的完整配置，装完就有文件树、模糊搜索、语法高亮、LSP 补全、Git 集成，直接可以用。

<!-- more -->

---

## Vi、Vim、Neovim、LazyVim 的关系

这几个名字经常混用，理清一下：

- **Vi**：1976 年诞生于 Unix，是最早的终端模态编辑器，确立了 Normal/Insert 模式这套操作体系。现在 macOS 自带的 `vi` 命令实际上指向的是 Vim。
- **Vim**（Vi IMproved）：1991 年发布，在 Vi 基础上加了语法高亮、多级撤销、插件系统等功能，成为几十年来最主流的终端编辑器。
- **Neovim**：2014 年从 Vim 分叉出来，目标是清理 Vim 几十年积累的历史包袱，加入原生 LSP 支持、异步任务、Lua 配置接口。现在活跃维护，插件生态比 Vim 更现代。
- **LazyVim**：不是一个独立软件，而是运行在 Neovim 上的一套配置框架。它把插件管理、LSP 配置、键位方案打包在一起，让你跳过从零配置的过程。

用一句话概括：LazyVim 是 Neovim 的配置，Neovim 是 Vim 的现代重写，Vim 是 Vi 的增强版。键位操作从 Vi 时代延续至今，学了 Neovim 就等于学了整个 Vi 系列。

---

## 为什么选 Neovim 而不是继续用 VSCode

不是说 VSCode 不好。但有几个场景 Neovim 明显更合适：

- SSH 连进服务器改配置文件，VSCode 的 Remote SSH 有时很慢甚至连不上
- 在终端里已经完成了 80% 的工作，不想为了改一个文件再打开 GUI
- 习惯了系列前几篇的工具之后，kitty 分屏 + Neovim 的工作流比 IDE 切换更顺

如果你只是想要一个顺手的编辑器，这篇文章也够用——LazyVim 装完就是一个功能完整的编辑器，不需要从头配置任何东西。

---

## [Neovim](https://neovim.io/) 安装

```bash
brew install neovim
```

验证安装：

```bash
nvim --version
# 应该显示 NVIM v0.12.x 或更高
```

---

## [LazyVim](https://www.lazyvim.org/) 安装：用别人调好的配置

LazyVim 是一套基于 lazy.nvim 插件管理器的 Neovim 配置框架，由 folke（lazy.nvim 作者）维护。安装方式是把它的配置仓库克隆到 Neovim 的配置目录。

### 前置依赖

LazyVim 依赖几个外部工具，系列前几篇已经装过其中大部分：

```bash
brew install git ripgrep fd lazygit
# ripgrep、fd：Telescope 文件搜索用
# lazygit：内置 Git TUI 集成
```

还需要一个 Nerd Font（用于图标显示）。如果用的是 kitty，需要给 kitty 配置 Nerd Font：

```bash
brew install --cask font-jetbrains-mono-nerd-font
```

然后在 `~/.config/kitty/kitty.conf` 里加一行：

```
font_family JetBrainsMono Nerd Font
```

### 备份已有配置（如果有的话）

```bash
mv ~/.config/nvim ~/.config/nvim.bak 2>/dev/null
mv ~/.local/share/nvim ~/.local/share/nvim.bak 2>/dev/null
mv ~/.local/state/nvim ~/.local/state/nvim.bak 2>/dev/null
```

### 克隆 LazyVim starter

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
# 去掉 .git 让这份配置成为你自己的，不绑定上游
```

### 首次启动

```bash
nvim
```

第一次打开会自动安装所有插件，等待进度条跑完（需要网络，几分钟）。安装完成后重启一次 nvim。

---

## Vim 最小生存指南

LazyVim 基于 Vim 键位，没用过 Vim 的话先记这几个：

**模式切换**：

```
i       进入 Insert 模式（开始输入）
Esc     回到 Normal 模式
v       进入 Visual 模式（选择文本）
:       进入 Command 模式（输入命令）
```

**Normal 模式下移动**：

```
h j k l   左下上右（也可以用方向键）
w         跳到下一个单词开头
b         跳到上一个单词开头
0         跳到行首
$         跳到行尾
gg        跳到文件开头
G         跳到文件末尾
Ctrl+d    向下翻半页
Ctrl+u    向上翻半页
```

**基础编辑**：

```
dd        删除当前行
yy        复制当前行
p         粘贴到下方
u         撤销
Ctrl+r    重做
/pattern  向下搜索，n 跳到下一个，N 跳到上一个
```

**保存和退出**：

```
:w        保存
:q        退出
:wq       保存并退出
:q!       强制退出不保存
```

---

## LazyVim 核心功能

LazyVim 在 Vim 基础键位上层叠了一套以 `Space` 为 leader 键的快捷键体系。按 `Space` 停顿一下，屏幕下方会弹出 which-key 提示窗口，显示所有可用的下一步操作。

### 文件操作

```
Space + e       打开/关闭文件树（neo-tree）
Space + f + f   模糊搜索文件名（Telescope）
Space + f + g   搜索文件内容（ripgrep）
Space + f + r   最近打开的文件
```

在文件树里：

```
a       新建文件
d       删除文件
r       重命名
Enter   打开文件
```

### LSP 功能（需要对应语言的 language server）

LazyVim 内置了 mason.nvim 来管理 LSP。打开一个 Go 或 Python 文件时会自动提示安装对应的 language server。

```
gd      跳到定义（go to definition）
gr      查看所有引用（go to references）
K       悬浮显示文档（hover doc）
Space + c + a   代码操作（code action，如自动 import）
Space + c + r   重命名符号
```

### 多窗口

```
Space + |   垂直分割窗口
Space + -   水平分割窗口
Ctrl+h/j/k/l    在分割窗口间移动
```

### Git 集成

```
Space + g + g   打开 lazygit（需要已安装）
Space + g + b   查看当前行的 git blame
```

---

## 常用快捷键速查

| 操作 | 快捷键 |
|------|--------|
| 查找文件 | `Space f f` |
| 搜索内容 | `Space f g` |
| 文件树 | `Space e` |
| 关闭当前 buffer | `Space b d` |
| 格式化文件 | `Space c f` |
| 跳到定义 | `gd` |
| 查看引用 | `gr` |
| 打开 lazygit | `Space g g` |
| 查看诊断（错误） | `Space x x` |
| 命令面板 | `Space :` |

按 `Space` 后等待 which-key 浮窗，是学习新快捷键最快的方式。

---

## 从哪里开始

不建议一上来就把 Neovim 当主力编辑器，学习曲线会很陡：

1. 先在低风险的场景用——改配置文件、看日志、临时编辑一个文件
2. 遇到不会的操作就按 `Space` 看 which-key 提示，或者 `:Tutor` 运行内置教程
3. 稳定用了一两周之后，再考虑迁移更重要的工作

LazyVim 的自定义配置放在 `~/.config/nvim/lua/plugins/` 下，新建一个 `.lua` 文件即可覆盖默认设置或添加插件，不需要改动 LazyVim 本身的文件。官方文档在 [lazyvim.org](https://www.lazyvim.org)。
