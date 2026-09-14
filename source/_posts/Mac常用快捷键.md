---
title: "Mac 常用快捷键"
date: 2026-09-14 19:41:00
categories: tools
tags:
  - Mac
  - macOS
  - 快捷键
  - 效率工具
cover: /img/p74.jpg
description: "Mac 键盘上没有 Home/End/Page Up/Page Down，这些功能其实都在，只是要靠组合键按出来。按分类整理常用快捷键，外加截屏和录屏。"
---

Mac 键盘上找不到 Home、End、Page Up、Page Down 这几个键，但功能都还在，只是苹果把它们塞进了 `Fn` 组合键里，键帽上完全看不出来。这篇按分类整理常用快捷键，条目都对照 [苹果官方支持文档](https://support.apple.com/en-us/102650) 核对过。

<!-- more -->

## 光标移动快捷键

1. `Fn + ←`：Home，跳到文档开头
2. `Fn + →`：End，跳到文档末尾
3. `Fn + ↑`：Page Up，向上翻一页
4. `Fn + ↓`：Page Down，向下翻一页
5. `Fn + Delete`：正向删除，删掉光标右边的字符（Mac 的 Delete 键默认只能删左边，也就是 Windows 键盘上的 Backspace）
6. `Command + ←` / `Command + →`：跳到当前行的行首 / 行尾
7. `Command + ↑` / `Command + ↓`：跳到整篇文档的开头 / 结尾
8. `Option + ←` / `Option + →`：按词跳转，跳到左边 / 右边那个词
9. `Option + Delete`：删除光标左边一整个词
10. `Control + A` / `Control + E` / `Control + K`：来自 Emacs 编辑器的一套光标操作（行首/行尾/剪切到行尾），几乎所有系统文本框都能用，完整版见 [命令行编辑进阶：Emacs 快捷键与甩进 vim 编辑](/命令行编辑进阶-Emacs快捷键与甩进vim编辑)

## 应用与窗口切换快捷键

1. `Command + Tab`：切换到另一个应用
2. `` Command + ` ``：切换同一个应用内的不同窗口（重音符键，一般在 Tab 键上方）
3. `Option + Command + Esc`：强制退出没反应的应用
4. `Command + H`：隐藏当前应用
5. `Option + Command + H`：隐藏"其他"所有应用，只留当前这一个
6. `Command + M`：最小化当前窗口
7. `Option + Command + M`：最小化当前应用的所有窗口
8. `Fn + H`（或 `Fn + F11`）：显示桌面，隐藏所有窗口
9. `Control + ↑`：打开 Mission Control，看到所有窗口
10. `Control + ↓`：只显示当前应用的所有窗口

## Finder 快捷键

1. `Command + Delete`：把选中文件移到废纸篓
2. `Shift + Command + Delete`：清空废纸篓（弹确认框）
3. `Option + Shift + Command + Delete`：清空废纸篓，不弹确认框
4. `Shift + Command + G`：打开"前往文件夹"，手动输入路径直达
5. `Shift + Command + A`：直达"应用程序"文件夹
6. `Shift + Command + U`：直达"实用工具"文件夹
7. `Option + Command + L`：直达"下载"文件夹
8. `Command + Shift + .`（句点）：显示或隐藏 Finder 里的隐藏文件
9. `Option + Command + V`：配合 `Command + C` 使用，效果是剪切并移动文件，不是复制一份
10. `Space`：快速查看选中的文件

## 系统与菜单栏快捷键

1. `Control + Command + Q`：锁屏
2. `Command + Space`：打开或关闭 Spotlight 搜索
3. `Control + Command + Space`（或 `Fn + E`）：打开表情符号选择器
4. `Control + Command + D`：查看选中单词的词典释义
5. `Command + ,`（逗号）：打开当前 App 的偏好设置
6. `Option +` 点击菜单栏 Wi-Fi 图标：直接看到 IP 地址、路由器地址、信道等详细信息
7. `Option +` 点击菜单栏亮度/音量图标：直接打开对应的设置面板

## 截屏快捷键

1. `Shift + Command + 3`：截取整个屏幕
2. `Shift + Command + 4`：截取选定区域，拖拽选取范围（按住 `Space` 可整体移动选区，`Esc` 取消）
3. `Shift + Command + 4`，再按 `Space`：截取单个窗口或菜单（按住 `Option` 点击可排除窗口阴影）
4. `Command + Control + Shift + 3`（或 `4`）：截图直接进剪贴板，不存文件，截完直接 `Command + V` 粘贴

## 录屏快捷键

1. `Shift + Command + 5`：打开截图/录屏工具栏，里面能选"录制整个屏幕""录制选中窗口"（macOS Tahoe 26 及以后）"录制选定区域"
2. 工具栏里点 `Options`：设置麦克风、鼠标点击提示、录制倒计时、保存位置，Tahoe 26 及以后还能选 SDR/HDR 格式
   - `SDR`（标准动态范围）：用 H.264 编码，文件更通用，随便什么设备、App 都能正常打开播放
   - `HDR`（高动态范围）：用 HEVC 编码，画面亮部和暗部的细节、色彩都保留得更多，但只有支持 HDR 的屏幕和播放器才能显示出这个效果，普通屏幕上跟 SDR 看不出明显差别，文件也更大
3. `Command + Control + Esc`：停止当前的屏幕录制
4. `Control + Command + N`：在 QuickTime Player 里直接新建屏幕录制，不用先点菜单
