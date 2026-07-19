---
title: "tmux 教程：SSH 断线不丢任务、分屏与会话管理"
date: 2026-07-19 23:51:32
cover: /img/p56.jpg
categories: tools
tags:
  - Linux
  - tmux
  - 终端
  - 效率工具
description: "tmux 让 SSH 断线不丢任务：session 分离与恢复、window/pane 分屏、滚屏与鼠标配置。"
---

SSH 上服务器跑了两个小时的数据迁移，网一抖连接断了——任务跟着一起被杀死，从头再来。问题的根源是：普通终端里，你跑的所有程序都挂在这条 SSH 连接下面，连接断了它们就成了孤儿被系统清理。tmux 解决的正是这件事：把"干活的会话"和"网络连接"解耦，连接断了会话照常活着。

<!-- more -->

这是终端工具系列的一篇，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)。

## [tmux](https://github.com/tmux/tmux/wiki) 是什么：终端复用器与三层概念

tmux（terminal multiplexer，终端复用器）在终端和你跑的程序之间加了一层"会话服务器"：程序挂在 tmux 的会话下而不是 SSH 连接下，所以连接断开、终端关闭都不影响它们继续跑。装起来一句话：`apt install tmux`（Linux）或 `brew install tmux`（macOS）。

它的界面组织分三层，从大到小：

```text
session（会话）    一个独立的工作环境，可以整体分离和恢复——tmux 的灵魂
  └─ window（窗口）  会话里的"标签页"，一个会话可以开多个，像浏览器 tab
      └─ pane（窗格）  窗口再切分出的小块屏幕，一屏同时看多个命令的输出
```

日常最常用的是 session 这一层（保命）和 pane 这一层（分屏），window 介于两者之间按需使用。

## 核心场景：断线不丢任务

先把最值钱的工作流走一遍：

```bash
tmux new -s migrate
#     新建一个名叫 migrate 的会话，进入后看到的还是普通 shell
#     -s：session 名字，起个有意义的名字，恢复时好认

./run-migration.sh
#     在会话里正常跑你的长任务

# 按 Ctrl-b 再按 d —— 分离（detach）会话
#     回到了原来的终端，但会话和里面的任务还在后台跑着
#     这时候断开 SSH、合上笔记本、下班，都不影响它

tmux ls
#     下次连上服务器，先列出所有活着的会话
#     migrate: 1 windows (created ...) —— 还活着

tmux attach -t migrate
#     -t：target，重新接上这个会话——屏幕内容、运行状态和离开时一模一样
```

两个补充命令：

```bash
tmux new -A -s work
#     -A：会话已存在就直接附加，不存在才新建——写进登录脚本很合适，
#     每次连服务器自动进入同一个工作环境

tmux kill-session -t migrate
#     任务彻底完了，删掉这个会话
```

## prefix 前缀键：所有快捷键的入口

tmux 的快捷键都是"两段式"：先按前缀键 `Ctrl-b`（按完松开），再按功能键。前面用到的分离就是 `Ctrl-b` 然后 `d`。忘了某个键绑定时，`Ctrl-b ?` 列出全部快捷键，`q` 退出查看。

下文的快捷键都省略前缀，写作"`Ctrl-b` + 键"。

## window：会话里的标签页

```text
Ctrl-b c        新建窗口（create）
Ctrl-b ,        重命名当前窗口——底部状态栏会显示名字，多窗口时不迷路
Ctrl-b 0-9      按编号直接跳转
Ctrl-b n / p    下一个 / 上一个窗口（next/previous）
Ctrl-b w        所有窗口的列表预览，回车跳转
Ctrl-b &        关闭当前窗口（会确认）
```

典型用法：窗口 0 跑服务、窗口 1 看日志、窗口 2 敲命令，`Ctrl-b 0/1/2` 来回跳。

## pane：一屏切多块

```text
Ctrl-b %        左右分屏（竖着切一刀）
Ctrl-b "        上下分屏（横着切一刀）
Ctrl-b 方向键    在窗格之间移动光标焦点
Ctrl-b z        放大当前窗格到全屏，再按一次恢复分屏布局（zoom）
Ctrl-b x        关闭当前窗格（会确认）
Ctrl-b q        闪出每个窗格的编号，接着按数字可跳过去
```

`%` 和 `"` 这两个键位不太直观，可以这么记：`%` 这个符号中间是一条斜线把两个圈分开——竖着切；`"` 是两个点并排躺着——横着切。

`Ctrl-b z` 值得单独点名：分屏跑着四块日志，想凑近看其中一块，`z` 放大到全屏，看完再 `z` 弹回原布局——比关掉重开窗格优雅得多。

## 滚屏问题：为什么鼠标滚轮不管用

新手进 tmux 的第一个困惑：想往上翻看输出，鼠标滚轮却不动历史内容。因为 tmux 默认不把滚轮事件交给历史缓冲区，翻历史要进 copy mode：

```text
Ctrl-b [        进入 copy mode，此时可以用方向键 / PgUp / PgDn 翻历史
q               退出 copy mode，回到实时画面
```

不想记这个的话，一行配置让鼠标直接可用（推荐）：

```bash
# 写进 ~/.tmux.conf
set -g mouse on
#     滚轮翻历史、点击切换窗格、拖动调窗格大小全部生效
```

## 最小可用配置：~/.tmux.conf

tmux 开箱即用，配置文件不是必需的；但这两行值得一开始就加上：

```bash
set -g mouse on
#     鼠标支持，上一节说过

set -g base-index 1
#     窗口编号从 1 开始而不是 0——Ctrl-b 1 比 Ctrl-b 0 顺手，键盘上 1 也更近
```

改完配置后，在 tmux 里执行 `tmux source-file ~/.tmux.conf` 立即生效，不用重启会话。

## 命令行操作速查表

| 命令 | 作用 |
|---|---|
| `tmux new -s 名字` | 新建命名会话 |
| `tmux new -A -s 名字` | 有则附加、无则新建，适合写进登录脚本 |
| `tmux ls` | 列出所有会话 |
| `tmux attach -t 名字` | 附加到指定会话 |
| `tmux kill-session -t 名字` | 删除指定会话 |
| `tmux source-file ~/.tmux.conf` | 重新加载配置 |

## 快捷键速查表（均先按 Ctrl-b 前缀）

| 键 | 作用 |
|---|---|
| `d` | 分离会话，任务继续后台跑 |
| `?` | 列出全部快捷键 |
| `c` / `,` / `&` | 新建 / 重命名 / 关闭窗口 |
| `0-9` / `n` / `p` / `w` | 跳转 / 下一个 / 上一个 / 列表预览窗口 |
| `%` / `"` | 左右分屏 / 上下分屏 |
| 方向键 | 切换窗格焦点 |
| `z` | 当前窗格全屏/恢复切换 |
| `x` / `q` | 关闭窗格 / 显示窗格编号 |
| `[` | 进入 copy mode 翻历史，`q` 退出 |

## 个人推荐配置

```bash
# ~/.tmux.conf
set-option -g status-keys vi
setw -g mode-keys vi

setw -g monitor-activity on

# setw -g c0-change-trigger 10
# setw -g c0-change-interval 100

# setw -g c0-change-interval 50
# setw -g c0-change-trigger  75


set-window-option -g automatic-rename on
set-option -g set-titles on
set -g history-limit 100000

#set-window-option -g utf8 on

# set command prefix
#set-option -g prefix C-a
#unbind-key C-b
#bind-key C-a send-prefix

bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D

bind < resize-pane -L 7
bind > resize-pane -R 7
bind - resize-pane -D 7
bind + resize-pane -U 7


bind-key -n M-l next-window
bind-key -n M-h previous-window



set -g status-interval 1
# status bar
set -g status-bg black
set -g status-fg blue


#set -g status-utf8 on
set -g status-justify centre
set -g status-bg default
set -g status-left " #[fg=green]#S@#H #[default]"
set -g status-left-length 20


# mouse support
# for tmux 2.1
# set -g mouse-utf8 on
set -g mouse on
#
# for previous version
#set -g mode-mouse on
#set -g mouse-resize-pane on
#set -g mouse-select-pane on
#set -g mouse-select-window on


#set -g status-right-length 25
set -g status-right "#[fg=green]%H:%M:%S #[fg=magenta]%a %m-%d #[default]"

# fix for tmux 1.9
bind '"' split-window -vc "#{pane_current_path}"
bind '%' split-window -hc "#{pane_current_path}"
bind 'c' new-window -c "#{pane_current_path}"

# run-shell "powerline-daemon -q"

# vim: ft=conf
```



## 写到这里

tmux 的学习路径建议按价值排序：先把"new → 干活 → d 分离 → attach 恢复"这条保命链路焊进肌肉记忆（[SSH](/SSH免密登录与scp实战-密钥配置与传文件) 断线从此只是小插曲），再加上 `set -g mouse on` 解决滚屏，最后按需拾起分屏快捷键。终端环境本身的配置（模拟器、shell 框架）另见 [kitty + Zim 那篇](/Mac终端基础-kitty和Zim)，和 tmux 是互补的两层。
