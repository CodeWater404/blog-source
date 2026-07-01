---
title: "yazi：终端里的三栏文件管理器"
date: 2026-07-01 14:00:00
cover: /img/p15.jpg
categories: tools
tags:
  - Mac
  - CLI
  - Terminal
  - 效率工具
description: "Mac 终端文件管理器 yazi 完整教程：安装与依赖配置、核心快捷键速查、kitty 图片预览配置、退出自动 cd 的 y 函数，以及插件安装方法。"
---

终端里管理文件，`cd` + `ls` 能用，但一旦目录层级深了，或者需要同时看父目录和当前目录，就很别扭。yazi 是用 Rust 写的终端文件管理器，三栏布局（父目录 / 当前目录 / 预览），操作用 Vim 键位，代码预览靠 bat，图片在 kitty 下能内联显示。

<!-- more -->

---

## [yazi](https://yazi-rs.github.io/)：三栏布局 + 异步架构

界面分三栏：左边是父目录，中间是当前目录，右边是文件预览。移动到哪个文件，右边实时更新预览内容：代码文件语法高亮（靠 bat），图片在 kitty 下直接渲染，普通文本直接显示。

和老牌的 ranger 比：ranger 是 Python 写的，启动稍慢，配置更复杂；yazi 是 Rust + 完全异步 I/O，大目录不会卡顿，插件用 Lua 写，生态也在快速成长。

---

## 安装：先确认依赖

yazi 的代码预览、文件搜索功能依赖之前装过的工具：

| 依赖 | 用途 |
|------|------|
| bat | 代码文件语法高亮预览 |
| fd | 文件名搜索（`s` 键触发） |
| ripgrep | 文件内容搜索（`S` 键触发） |

如果前三篇已经装过 bat/fd/rg，直接装 yazi 就行：

```bash
brew install yazi
```

安装完后 `ya` 命令是插件管理器，`yazi` 命令启动主程序：

```bash
yazi --version
# Yazi 26.5.6
```

---

## 三分钟上手：核心快捷键

启动：

```bash
yazi
```

### 导航

| 按键 | 操作 |
|------|------|
| `h` | 返回父目录 |
| `l` | 进入子目录 / 打开文件 |
| `j` / `k` | 下移 / 上移 |
| `g g` | 跳到列表顶部 |
| `G` | 跳到列表底部 |

### 文件操作

| 按键 | 操作 |
|------|------|
| `Space` | 多选（切换选中状态） |
| `y` | 复制选中文件 |
| `x` | 剪切选中文件 |
| `p` | 粘贴 |
| `d` | 移动到回收站 |
| `D` | 永久删除（不可恢复） |
| `a` | 新建文件（以 `/` 结尾则新建目录） |
| `r` | 重命名 |
| `.` | 切换隐藏文件显示（`.` 开头的文件） |

### 搜索与过滤

| 按键 | 操作 |
|------|------|
| `f` | 按名称过滤（在当前目录实时筛选） |
| `/` | 向下查找文件（按名称匹配，`n` 跳下一个） |
| `s` | 按文件名搜索（调用 fd，可跨目录） |
| `S` | 按内容搜索（调用 ripgrep） |
| `Ctrl + s` | 取消正在进行的搜索 |
| `z` | 用 fzf 跳转目录 |
| `Z` | 用 zoxide 跳转目录 |

### 其他

| 按键 | 操作 |
|------|------|
| `Enter` | 打开文件（调用系统默认程序） |
| `q` | 退出 |
| `F1` / `~` | 查看所有快捷键（帮助菜单） |

---

## kitty 下启用图片内联预览

yazi 支持多种图片协议，在 kitty 终端下效果最好，直接把图片渲染在预览面板里，不需要额外安装 ImageMagick。

kitty 协议是 yazi 的默认优先选项，如果你用的是 kitty，装完即用，不需要额外配置。打开 yazi，导航到任意图片文件，右侧预览面板就会显示图片。

如果图片没有显示，检查 `TERM` 环境变量：

```bash
echo $TERM
# 应该输出 xterm-kitty
```

输出是 `xterm-kitty` 说明 kitty 协议生效。如果不是，可能是在嵌套终端或 tmux 里启动了 yazi，协议传递受阻。

---

## .zshrc 集成：退出时自动 cd

直接运行 `yazi` 有个问题：退出后，终端还停在进入 yazi 前的目录，而不是你在 yazi 里最后浏览的位置。

用 `--cwd-file` 参数可以解决，在 `~/.zshrc` 里加这个函数：

```bash
function y() {
    local tmp="$(mktemp -t "yazi-cwd")"
    yazi "$@" --cwd-file="$tmp"
    #          --cwd-file：退出时把当前目录路径写入指定文件
    if cwd="$(cat -- "$tmp")" && [ -n "$cwd" ] && [ "$cwd" != "$PWD" ]; then
        cd -- "$cwd"
    fi
    rm -f -- "$tmp"
}
```

之后用 `y` 命令启动，退出 yazi 时 shell 会自动 `cd` 到你最后停留的目录。`yazi` 命令照常可以用，行为不变。

---

## 插件安装

yazi 用 `ya pkg` 命令管理插件，插件托管在 GitHub 上，格式是 `用户名/仓库名:插件名`。

```bash
# 添加插件（写入 package.toml，同时下载）
ya pkg add yazi-rs/plugins:zoom
#          yazi-rs/plugins:zoom：仓库:插件名（一个仓库可以包含多个插件）

# 安装 package.toml 里记录的所有插件（换机器后用）
ya pkg install

# 更新所有插件
ya pkg upgrade

# 列出已安装插件
ya pkg list
```

装完插件后还需要在 `~/.config/yazi/keymap.toml` 里绑定快捷键才能用。

**几个实用插件**：

| 插件 | 仓库 | 功能 |
|------|------|------|
| zoom | `yazi-rs/plugins:zoom` | 预览面板图片缩放（`=` 放大 / `-` 缩小） |
| toggle-pane | `yazi-rs/plugins:toggle-pane` | 切换全屏预览模式（`T` 键触发） |
| git | `yazi-rs/plugins:git` | 在文件列表里显示 Git 状态 |
| diff | `yazi-rs/plugins:diff` | 对比两个文件的差异 |

zoom 和 toggle-pane 是官方插件仓库里的，直接 `ya pack -a` 就能装。

---

## 常见问题

**yazi 和 ranger 的区别是什么，值得换吗？**

主要区别在性能和配置方式。ranger 是 Python 写的，大目录（几千个文件）下滚动会有延迟；yazi 是 Rust + 异步 I/O，大目录不卡。配置上，ranger 用 Python 写脚本，yazi 用 Lua 写插件。如果你现在用 ranger 没有痛点，不用急着换；如果有性能问题，或者已经用 kitty 希望图片预览更好，换 yazi 值得试试。

**图片预览一直显示不出来怎么办？**

按顺序排查：1）确认用的是 kitty 终端，`echo $TERM` 输出应该是 `xterm-kitty`；2）如果在 tmux 里，需要在 `~/.tmux.conf` 里加以下 3 行，然后重启 tmux：

```bash
set -g allow-passthrough on
set -ga update-environment TERM
set -ga update-environment TERM_PROGRAM
```

3）非 kitty 终端（如 iTerm2、WezTerm）也支持内联图片，yazi 会自动检测，无需额外配置。

**`y` 函数和直接运行 `yazi` 有什么区别？**

`y` 函数是对 `yazi` 的包装，核心差别只有一个：退出后是否自动 `cd`。`yazi` 退出后目录不变，`y` 退出后自动跳到你在 yazi 里最后浏览的目录。传参方式一样，`y ~/Documents` 等同于 `yazi ~/Documents`，直接在指定目录里打开 yazi。
