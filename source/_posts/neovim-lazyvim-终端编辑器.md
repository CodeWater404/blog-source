---
title: "从零开始用 Neovim：LazyVim 让配置不再是障碍"
date: 2026-07-03 18:00:00
updated: 2026-09-21 19:24:20
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
# ripgrep、fd：文件名和内容搜索用
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
Space + f + f   模糊搜索文件名，范围是项目根目录
Space + f + F   同上，但范围是当前工作目录（cwd）
Space + f + g   只在 git 跟踪的文件里找文件名
Space + f + r   最近打开过的文件，所有目录都算
Space + f + R   最近打开过的文件，只算当前工作目录下的
Space + s + g   搜索文件内容（ripgrep），范围是项目根目录
Space + s + G   同上，但范围是当前工作目录
```

这里的"项目根目录"和"当前工作目录"不是一回事：

1. 当前工作目录（cwd）：你在终端里执行 `nvim` 时所在的目录，在 Neovim 里用 `:pwd` 可以看到。
2. 项目根目录：LazyVim 按这个顺序找，找到哪个用哪个：当前文件所属 LSP 的工作区根目录 → 往上找到的第一个含 `.git` 或 `lua` 的目录 → 都没有就退回 cwd。

比如在项目的 `src/handler/` 目录下启动 `nvim`，`Space f f` 搜的是整个仓库，`Space f F` 只搜 `src/handler/` 下面。

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

`gd` 跳过去之后想跳回来，按 `Ctrl-o` 就行——这是 vim 自带的跳转列表机制，不是 LazyVim 专门配的，具体原理见 [vim 基本操作](/vim基本操作-模式切换到保存退出)"常见操作场景"一节。

### 多窗口

```
Space + |   垂直分割窗口
Space + -   水平分割窗口
Ctrl+h/j/k/l    在分割窗口间移动
```

### 会话：退出后接着上次的工作继续

[persistence.nvim](https://github.com/folke/persistence.nvim) 是 LazyVim 自带的会话管理插件。默认行为是：你退出 nvim 时，它自动把当前目录的窗口布局、打开的文件存成一个会话文件；下次需要时，一个快捷键恢复。底层用的就是 vim 原生的 `:mksession`，原理见 [vim 基本操作](/vim基本操作-模式切换到保存退出)"常见操作场景"一节。

```
Space + q + s   恢复当前目录的会话
Space + q + l   恢复最近一次保存的会话，不限目录
Space + q + S   弹出列表，选一个目录的会话恢复
Space + q + d   这次退出时不保存会话（之前存过的旧会话不会被删）
Space + q + q   退出全部窗口，等于 :qa
```

`Space q` 这一组在 which-key 里叫 `quit/session`，退出和会话都归它管，所以 `Space q q` 也在里面。不带文件启动 `nvim` 时，仪表盘上的 `s`（Restore Session）等价于 `Space q s`。

会话是这样存的：

1. 触发时机：退出 nvim 时自动保存，不需要手动操作。前提是至少打开了一个真实文件，只有空缓冲区、git commit 编辑窗口这类不会存。
2. 存放位置：`~/.local/state/nvim/sessions/`，文件名是工作目录把 `/` 换成 `%`，比如 `%Users%me%code%api.vim`。每个目录一份。
3. 按 git 分支区分：在 `main`、`master` 以外的分支上，文件名末尾会多一段分支名，每个分支各存一份。这个分支还没存过会话时，`Space q s` 会退回去用不带分支名的那份。
4. 存的内容：LazyVim 配置的 `sessionoptions` 包括打开的文件、窗口和标签页的布局与大小、工作目录、折叠状态等，只存文件路径，不存文件内容。

### Git 集成

```
Space + g + g   打开 lazygit（需要已安装）
Space + g + b   查看当前行的 git blame
```

### 跳到屏幕上任意位置：flash.nvim

vim 自带的 `f`/`t` 只能在当前这一行里跳（见 [vim 基本操作](/vim基本操作-模式切换到保存退出)"行内跳转"一节）。LazyVim 自带 [flash.nvim](https://github.com/folke/flash.nvim)：输入几个字符后，屏幕上所有匹配的位置都会冒出标签字母，按标签就跳过去，不用再一路 `j`/`k`/`w` 地挪。

1. `s`：输入要找的字符，想输几个都行，输得越多匹配越少；按匹配处的标签字母跳过去，按 `Enter` 直接跳到第一个匹配。分割出来的其他窗口里的匹配也会显示标签。
2. `S`：Flash Treesitter，给光标所在语法节点的各层父节点标上标签（比如某个参数、整个函数），选标签就选中那一层，适合整块选中一段代码。
3. `Ctrl-s`：在普通的 `/` 搜索过程中按，开关搜索结果上的跳转标签。
4. `yr`（`dr`、`cr` 同理）：远程操作。按 `yr` 后先选标签跳到远处某个位置，再输入一个 motion（比如 `iw`），复制的是远处那个词，完成后光标自动回到原来的位置，不用跳过去再跳回来。

注意 `s` 原本是 vim 的"删除一个字符并进入插入模式"，装了 flash 之后被占用了，想要原来的效果用 `cl`。

---

## 常用快捷键速查

| 操作 | 快捷键 |
|------|--------|
| 查找文件 | `Space f f` |
| 搜索内容 | `Space s g` |
| 文件树 | `Space e` |
| 关闭当前 buffer | `Space b d` |
| 格式化文件 | `Space c f` |
| 跳到屏幕上任意位置 | `s` |
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
