---
title: "命令行编辑进阶：Emacs 快捷键、甩进 vim 编辑命令行与跳转到指定行"
date: 2026-07-13 00:30:02
cover: /img/p39.jpg
categories: tools
tags:
  - Linux
  - Emacs
  - readline
  - 命令行
  - CLI
description: "bash/zsh 默认是 Emacs 键位：Ctrl-A/E/W/U/K/Y 怎么用、Ctrl-X Ctrl-E 甩进编辑器改命令，附 vim 跳转指定行技巧。"
---

命令敲到一半打错了个字符，很多人的反应是按住 `←` 挪半天光标，或者干脆全删了重打。其实 bash、zsh 这些 shell 默认就内置了一整套 Emacs 风格的编辑快捷键，光标跳词、整段删除、剪切粘贴全都有对应按键，不需要任何配置就能用。

这篇讲清楚这套快捷键怎么用，外加一个进阶技巧：命令太长太复杂的时候，直接把它甩进编辑器里改。

<!-- more -->

这是终端效率小系列第三篇，承接 [不熟悉的命令怎么快速搞懂](/不熟悉的命令怎么快速搞懂-man页面结构与apropos速查) 和 [vim 基本操作](/vim基本操作-模式切换到保存退出)——本篇最后一个技巧会直接用到上一篇学的 vim 保存退出操作。

## 为什么这套快捷键几乎所有 shell 都能用

命令行的编辑功能，bash 靠的是 GNU Readline 库，zsh 靠的是自己的 ZLE（Zsh Line Editor）。两者设计上的默认编辑模式都是 Emacs 风格——但这里有个很容易踩的坑：**zsh 有条隐藏规则，只要 `$EDITOR` 或 `$VISUAL` 的值里包含字符串 "vi"（`vim`、`nvim` 都算在内），zsh 启动时就会自动切换成 vi 键位，而不是 Emacs 键位**。开发者把 `$EDITOR` 设成 `vim` 或 `nvim` 是很常见的习惯，这种情况下 zsh 默认给的其实是 vi 键位，本文这些 Emacs 快捷键很可能大部分不生效或者行为完全对不上。

```bash
echo $EDITOR
#     先看一眼这个变量的值，只要包含 "vi"（vim、nvim 都算），zsh 大概率已经是 vi 键位了

bindkey -e
#     写进 ~/.zshrc，强制 zsh 使用 Emacs 键位，不再受 $EDITOR 内容影响

set -o vi
#     bash 里想切换成 vi 风格的编辑模式（用 hjkl 那一套）用这个
#     但切过去之后，本文这些 Emacs 快捷键大部分失效，两套模式二选一

set -o emacs
#     切回 Emacs 模式，bash 默认就是这个，不受 $EDITOR 影响——这点和 zsh 不一样
```

不确定自己当前是哪种键位，直接试一下 `Ctrl-A` 能不能跳到行首：能跳就是 Emacs 键位；没反应或者屏幕上蹦出一个奇怪符号，大概率是 vi 键位——vi 键位下得先按 `Esc` 进命令模式，`0` 才是跳到行首。

## 行内光标移动：不用方向键也能秒跳

```bash
Ctrl-A
#     跳到行首（A = ahead / 行的开头）

Ctrl-E
#     跳到行尾（E = end）

Alt-B
#     向左跳一个单词（B = back）

Alt-F
#     向右跳一个单词（F = forward）
```

`Alt-B`/`Alt-F` 在部分终端里按不出效果——很多终端默认不把 `Alt` 当 Meta 键处理，需要在终端设置里打开"Use Option as Meta Key"（macOS Terminal.app 的选项，iTerm2 类似）。打不开设置的话，用 `Esc` 代替 `Alt` 也一样：先按 `Esc` 松开，再按 `b` 或 `f`，效果等价于 `Alt-B`/`Alt-F`。

## 删除和剪切：Ctrl-W/U/K 存进 kill-ring，Ctrl-Y 取回

```bash
Ctrl-W
#     删除光标前面的一个单词（向后删）

Alt-D
#     删除光标后面的一个单词（向前删）——Ctrl-W 的反方向

Ctrl-K
#     删除从光标到行尾的所有内容

Ctrl-Y
#     粘贴——把上一次删除的内容粘贴回来（yank）
```

`Ctrl-U` 单独拎出来说，因为它在 bash 和 zsh 里行为不一样，是个真实存在的坑：bash 的 `Ctrl-U`（Readline 的 `unix-line-discard`）只删从光标到行首的部分，光标后面的内容原样保留；zsh 的 `Ctrl-U`（不管 vi 键位还是 Emacs 键位）绑定的都是 `kill-whole-line`，不管光标停在哪，直接清空整行。同一个按键，bash 和 zsh 下的效果是不同的，用 zsh 的人如果照着 bash 的经验去理解 `Ctrl-U`，会觉得"怎么光标后面的内容也被删了"。

```bash
Ctrl-U
#     bash：删除从光标到行首的内容，光标后面的保留
#     zsh：不管光标在哪，直接清空整行——这是 zsh 的 kill-whole-line，跟 bash 不是一回事
```

这几个删除命令背后有个共同点：删掉的内容不是直接消失，而是存进一个叫 kill-ring 的剪贴区，`Ctrl-Y` 粘贴的就是这里最新的一条。

## 清屏和历史命令反向搜索

```bash
Ctrl-L
#     清屏，光标当前这一行会被挪到清空后的屏幕顶部，命令内容不受影响

Ctrl-R
#     反向搜索历史命令，输入关键词实时匹配之前敲过的命令
#     再按一次 Ctrl-R 跳到更早的一条匹配，按 Enter 执行，按 Esc 只取用不执行
```

`Ctrl-R` 是历史命令搜索里最常用的一个，比按 `↑` 翻半天历史记录快得多——尤其是敲过成百上千条命令之后，翻方向键基本翻不到想要的那条。

## 进阶技巧：把命令行甩进编辑器里改

`Ctrl-A`/`Ctrl-E` 这类单字符编辑对付一行命令够用，但遇到又长又复杂的命令——比如带了好几个换行的多行脚本片段，或者一长串 JSON——在命令行里用方向键和退格键改，效率很低。这种场景该用 `Ctrl-X Ctrl-E`：

```bash
Ctrl-X Ctrl-E
#     把当前命令行的内容甩进编辑器，用完整的编辑器能力改
#     改完保存退出，编辑结果自动回填到命令行——不会直接执行，只是替换命令行内容
```

用哪个编辑器由 `$VISUAL`、`$EDITOR` 这两个环境变量决定，Bash 官方文档写得很明确：依次尝试 `$VISUAL`、`$EDITOR`，两个都没设置的话，最后退回 `emacs`：

```bash
export EDITOR=vim
#     把默认编辑器设成 vim，Ctrl-X Ctrl-E 打开的就是 vim
#     没设置这个变量时，bash 会依次尝试 $VISUAL、$EDITOR，都没有就用 emacs
```

进了编辑器之后，改完用上一篇学的 `:wq` 或 `:x` 保存退出（这正是本文系列第二篇的用武之地），命令行会自动变成编辑后的内容，回车才会真正执行。

有个地方要注意：这个功能在 bash 里是 Readline 自带的，默认已经绑定在 `Ctrl-X Ctrl-E` 上，装好 bash 就能用；但 zsh 不一样，ZLE 默认编辑模式虽然也是 Emacs，这个"甩进编辑器"的功能却不是默认绑定的，得在 `~/.zshrc` 里手动开启：

```bash
autoload -Uz edit-command-line
#     加载 zsh 自带的 edit-command-line 功能模块

zle -N edit-command-line
#     把它注册成一个 ZLE 可用的编辑命令

bindkey '^x^e' edit-command-line
#     绑定到 Ctrl-X Ctrl-E，跟 bash 保持一致的按键习惯
```

写完存好 `.zshrc`，`source ~/.zshrc` 或者重开一个终端窗口生效。

## vim/nvim 跳转到指定行

编辑长文件时，与其打开后再一步步找到目标行，不如打开的时候就直接带上跳转目标：

```bash
vim +42 access.log
#     打开文件后直接跳到第 42 行，光标停在那一行

vim +/ERROR access.log
#     打开文件后跳到第一个匹配 "ERROR" 的位置，等价于打开后手动 /ERROR 回车
```

`+42` 是 `+:42` 的简写形式——vim 允许在命令行用 `+{Ex命令}` 指定打开文件后自动执行的命令，`42` 本身就是一个合法的行号跳转 Ex 命令；`+/pattern` 同理，等价于打开后立刻执行一次 `/pattern` 搜索。

如果已经在 vim 里面了，跳转用 `:42` 或者 `42G`——后者就是[上一篇 vim 文章](/vim基本操作-模式切换到保存退出)里提到的 `G` 命令，前面加数字表示跳到第几行，和 `gg`/`G` 是同一套逻辑的延伸。

## 终端效率小系列：写到这里

三篇写下来，从 [不熟悉的命令怎么快速搞懂](/不熟悉的命令怎么快速搞懂-man页面结构与apropos速查)，到 [vim 的基本操作](/vim基本操作-模式切换到保存退出)，再到这篇 Emacs 风格的命令行编辑——串起来就是一套完整的终端使用习惯：查文档、编辑文件、编辑命令行，三件事都能用最少的按键完成。剩下的就是练：这些快捷键只有敲进肌肉记忆之后才会真正提速，光看不练效果有限。
