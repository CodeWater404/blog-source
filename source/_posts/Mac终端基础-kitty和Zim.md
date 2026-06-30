---
title: "Mac 终端基础配置：kitty + Zim"
date: 2026-06-30 10:00:00
categories: tools
tags:
  - Mac
  - CLI
  - Terminal
  - kitty
  - Zim
  - zsh
description: "从零开始配置 Mac 终端环境：用 GPU 加速的 kitty 替换默认终端，用 Zim 框架升级 zsh，配齐语法高亮、自动补全、fzf 模糊补全等插件。"
---

macOS 默认终端在高分屏上滚动大量输出会掉帧，Oh My Zsh 每次开终端要等一两秒。换上 kitty + Zim 之后，这两个问题都消失了。

这篇讲怎么装、怎么配。后续所有工具（eza、yazi、fzf 等）都运行在这个基础上，建议先把这两个配好。

<!-- more -->

---

## [kitty](https://sw.kovidgoyal.net/kitty/)：GPU 渲染的终端模拟器

### 为什么换掉默认终端

macOS 自带的 Terminal.app 够用，但渲染靠 CPU，在高分屏上滚动大量输出时会掉帧。iTerm2 是很多人的选择，但功能过于复杂，配置项太多。kitty 用 GPU 渲染，启动和滚动都很快，配置文件是单个 `kitty.conf`，改起来直接。

### 安装 kitty

```bash
brew install --cask kitty
```

装完从 Launchpad 或直接 `open -a kitty` 启动。

### 配置文件位置

kitty 的配置文件在 `~/.config/kitty/kitty.conf`。第一次启动后这个文件会自动生成，里面全是注释和默认值，实际生效的配置只需要在文件末尾追加你要改的项就行，不需要动前面的默认注释。

用 kitty 自身打开配置文件：

```bash
# kitty 内置了编辑配置的快捷键
# 按 Ctrl+Shift+F2 会直接用默认编辑器打开 kitty.conf
# 或者直接：
nvim ~/.config/kitty/kitty.conf
```

### 我的实际配置

下面是配置文件末尾追加的内容，注释说明了每项的作用：

```conf
# 字体大小，默认 11.0 在高分屏上偏小
font_size 16.0

# 光标形状改为竖线（beam），比默认的方块光标更精确
cursor_shape beam

# 让 Option 键作为 Alt 键使用
# 不开这个，fzf 的 Alt+C 跳目录快捷键等无法生效
macos_option_as_alt yes

# 自定义快捷键：Ctrl+Alt+Z 将当前窗格全屏（stack 布局）
# kitty 内置分屏，这个快捷键快速聚焦单个窗格
map ctrl+alt+z toggle_layout stack

# 背景图配置（可选）
# 设置一张壁纸作为终端背景，background_tint 控制遮罩深度
# background_blur 是高斯模糊半径，让文字更易读
background_image ~/Pictures/wallpaper/your-wallpaper.jpg
background_image_layout scaled
background_image_linear yes
background_tint 0.7
background_opacity 0.8
background_blur 32
```

其中 `macos_option_as_alt yes` 这一项最关键，后续很多工具（fzf 的 `Alt+C`、Zsh 的 `Alt+.` 等）依赖 Alt 键，不开这个这些快捷键全都用不了。

### 字体：安装 Nerd Font

eza 显示文件图标、yazi 文件管理器、lazygit 等工具都依赖 [Nerd Font](https://www.nerdfonts.com/) 字体来渲染图标。推荐 JetBrains Mono Nerd Font：

```bash
brew install --cask font-jetbrains-mono-nerd-font
```

装好后在 `kitty.conf` 里指定：

```conf
font_family JetBrainsMono Nerd Font Mono
```

### 内置分屏

kitty 自带分屏功能，不需要另装 tmux：

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+Shift+Enter` | 向右新建窗格 |
| `Ctrl+Shift+]` / `[` | 切换到下/上一个窗格 |
| `Ctrl+Alt+Z` | 全屏当前窗格（上面配置的自定义键） |
| `Ctrl+Shift+T` | 新建标签页 |
| `Ctrl+Shift+Q` | 关闭当前窗格 |

### 重载配置

改完 `kitty.conf` 后不需要重启，按 `Ctrl+Shift+F5` 重载配置即可生效。

---

## [Zim](https://zimfw.sh/)：比 Oh My Zsh 快很多的 zsh 框架

### 为什么用 Zim 而不是 Oh My Zsh

Oh My Zsh 是最流行的 zsh 框架，但加载了大量默认插件，导致每次开终端都要等一两秒。Zim 只装你声明的插件，启动速度快很多，日常使用几乎感觉不到延迟。

### 安装 Zim

macOS 自带 zsh，直接安装 Zim 框架：

```bash
curl -fsSL https://raw.githubusercontent.com/zimfw/zimfw/master/install.zsh | zsh
```

这条命令会在 `~/.zshrc` 和 `~/.zimrc` 里写入初始化代码，然后自动安装默认插件。装完后重启终端或执行：

```bash
exec zsh
```

### 配置文件：.zimrc

Zim 的插件列表在 `~/.zimrc`，每行一个 `zmodule` 声明。改完后运行 `zimfw install` 安装新插件，`zimfw uninstall` 移除已删除的插件。

下面是完整的 `.zimrc` 配置：

```bash
# 基础模块
zmodule environment   # 设置合理的 zsh 内置选项
zmodule git           # git 常用别名（gst / gco / gp 等）
zmodule input         # 修正方向键、Ctrl 组合键的 bindkey
zmodule termtitle     # 在终端标题栏显示当前目录
zmodule utility       # 给 ls、grep、less 加颜色

# 补全
zmodule zsh-users/zsh-completions --fpath src  # 扩充补全定义
zmodule completion    # 启用 zsh 智能补全（必须放在补全定义之后）

# 语法高亮和历史（顺序固定，不能调换）
zmodule zdharma-continuum/fast-syntax-highlighting  # 命令行实时语法高亮
zmodule zsh-users/zsh-history-substring-search      # 上下箭头按前缀过滤历史
zmodule zsh-users/zsh-autosuggestions               # 灰色自动补全建议

# fzf 补全集成
zmodule Aloxaf/fzf-tab  # Tab 补全候选列表变成 fzf 模糊搜索界面

# Prompt 主题
zmodule git-info   # steeef 主题依赖这个
zmodule steeef     # Prompt 主题
```

> **注意顺序：** fast-syntax-highlighting 必须在 history-substring-search 之前，history-substring-search 必须在 autosuggestions 之前。`completion` 必须在所有补全定义模块之后。Zim 文档里有说明，这个坑踩过才知道。

改完 `.zimrc` 后执行：

```bash
zimfw install
exec zsh
```

### 插件效果说明

**fast-syntax-highlighting**

输入命令时实时变色：命令存在且路径正确显示绿色，命令不存在或参数有误显示红色。按回车前就能看到拼写错误。

**zsh-history-substring-search**

按 `↑` 不再是顺序翻历史，而是按当前已输入内容做前缀匹配。比如输入 `git` 再按 `↑`，只在历史里找 `git` 开头的命令。

**zsh-autosuggestions**

根据历史记录实时预测并在光标后显示灰色补全建议。按 `→` 或 `End` 接受整条建议，按 `Ctrl+→` 接受一个单词。

**fzf-tab**

需要先安装 fzf（第二篇会讲）。装好后按 Tab 补全不再弹出静态候选列表，而是进入 fzf 交互界面，可以用模糊搜索过滤候选项，用 `↑↓` 选择后按回车确认。

**steeef 主题**

Prompt 显示格式：`用户名@主机 路径 [git分支 ±变动]`。有未提交的改动时 git 分支旁会显示标记。

---

## 验证配置是否生效

配好后重启 kitty 终端，逐项检查：

```bash
# 确认 Zim 正确加载，会输出已安装模块列表
zimfw info
```

**语法高亮**：在终端里输入 `ls`，字体应该变绿；输入一个不存在的命令（比如 `foobarbaz`），应该变红。如果颜色没变，说明 fast-syntax-highlighting 没装上，重新跑 `zimfw install`。

**历史子串搜索**：输入 `git`，然后按 `↑`，应该只显示历史里 `git` 开头的命令，而不是顺序往上翻。

**自动建议**：输入任意一个你用过的命令的前几个字符，光标后应该出现灰色建议。按 `→` 接受。

**fzf-tab**：随便输一个目录路径然后按 Tab，应该弹出 fzf 交互界面而不是静态列表。如果没变，大概率是 fzf 还没装——这个在第二篇安装完 fzf 后自动生效，不用管。

---

## 装好之后，立刻能感受到差异的一件事

配完 Zim 之后，找一条你经常用的长命令（比如某个带一堆参数的 git 命令），输前几个字符，按 `↑`。原来要翻几十条历史才能找到的命令，现在两三下就出来了。

装完 fzf 后 fzf-tab 也会同时生效，那是第二篇的内容。
