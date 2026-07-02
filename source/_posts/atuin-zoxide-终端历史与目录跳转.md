---
title: "Shell 历史与目录跳转的现代替代：atuin + zoxide"
date: 2026-07-02 10:00:00
cover: /img/p05.jpg
categories: tools
tags:
  - Mac
  - CLI
  - Terminal
  - atuin
  - zoxide
  - 效率工具
description: "用 atuin 替换 shell 历史记录，获得可过滤的 Ctrl+R 搜索界面；用 zoxide 替换 cd，输入路径片段直接跳转。两个工具安装简单、收益立竿见影。"
---

每天重复输入的内容里，有两类最浪费时间：一是翻命令历史找之前跑过的那条带很多参数的 curl，二是手打一串层层嵌套的目录路径。

atuin 解决第一个，zoxide 解决第二个。

<!-- more -->

---

## [atuin](https://atuin.sh/)：把命令历史变成可搜索的数据库

默认的 zsh 历史记录是一个纯文本文件，`Ctrl+R` 只能按前缀匹配向上翻。atuin 把历史存进本地 SQLite 数据库，记录命令的执行时间、工作目录、退出码、耗时，然后提供一个交互式全文搜索界面。

### 安装

```bash
brew install atuin
```

### 集成到 .zshrc

在 `~/.zshrc` 末尾加入：

```bash
eval "$(atuin init zsh)"
```

重新加载配置：

```bash
source ~/.zshrc
```

之后按 `Ctrl+R` 就会进入 atuin 的搜索界面，而不是原来的 zsh 历史翻找。

### Ctrl+R 界面

进入界面后：

- 直接输入关键词做全文搜索，不需要记住命令开头是什么
- `↑` `↓` 方向键移动选中，`Enter` 执行，`Esc` 退出
- 支持按前缀过滤：

```bash
# 在搜索框里输入这些前缀来缩小范围
cwd:~/code     # 只看在 ~/code 目录下执行过的命令
exit:0         # 只看执行成功的命令（退出码为 0）
before:1h      # 只看 1 小时内执行的命令
after:2026-07-01  # 只看某日期之后的命令
```

实际工作中最常用的是 `cwd:` 过滤——在一个项目目录里搜，不会翻出其他项目的干扰结果。

### 历史同步（可选）

atuin 支持跨设备同步命令历史，数据经过端对端加密后上传到 atuin 的云服务（也可以自己搭）。有多台 Mac 的话同步后 `Ctrl+R` 能看到所有设备的历史。

注册并登录：

```bash
atuin register -u 用户名 -e 邮箱 -p 密码
atuin login -u 用户名
# 之后会提示输入密码和加密密钥（注册时自动生成，atuin key 查看）
atuin sync
```

不需要同步的话跳过这步，atuin 本地也完整可用。

---

## [zoxide](https://github.com/ajeetdsouza/zoxide)：让 cd 记住你去过哪里

zoxide 是 `cd` 的替代品，核心逻辑是 **frecency**（频率 × 最近访问时间）：你去过的目录都会被记录，之后输入路径关键词，zoxide 根据访问频率和时间自动匹配最可能的那个。

### 安装

```bash
brew install zoxide
```

### 集成到 .zshrc

```bash
eval "$(zoxide init zsh)"
```

默认会添加 `z` 和 `zi` 两个命令。如果想用 `z` 完全替换原来的 `cd`，可以用 `--cmd cd` 参数：

```bash
eval "$(zoxide init zsh --cmd cd)"
```

这样原来的 `cd` 命令也会指向 zoxide，不需要改任何习惯。

### z 和 zi

`z` 直接跳转，`zi` 调出 fzf 交互界面选择：

```bash
z blog          # 直接跳到访问频率最高的包含 "blog" 的目录
z code blog     # 多个关键词同时过滤，跳到同时含 "code" 和 "blog" 的目录
zi              # 打开 fzf 界面，列出所有记录过的目录，可以模糊搜索
zi blog         # 带关键词预过滤后再打开 fzf 界面
```

`zi` 需要系统里装了 fzf，前几篇文章里已经装过了，直接可用。

### 路径是怎么学习的

第一次使用时 zoxide 没有任何记录，`z` 命令找不到目标会报错。正常用 `cd` 进入目录之后，zoxide 自动在后台记录。进过 10 个左右的目录之后，`z` 就开始有用了。也可以手动添加：

```bash
zoxide add ~/code/github/blog-source
```

查看当前数据库里记录了哪些目录以及对应的权重：

```bash
zoxide query --list --score
```

---

## .zshrc 完整配置

把两个工具的初始化放到 `.zshrc` 的末尾：

```bash
# atuin：增强命令历史，接管 Ctrl+R
eval "$(atuin init zsh)"

# zoxide：智能目录跳转，替换 cd（需在 compinit 之后）
eval "$(zoxide init zsh --cmd cd)"
```

顺序无所谓，两者互不依赖。

---

## 从哪里开始

这两个工具的上手成本都极低，装完改一行配置就生效，不需要专门学习：

- **atuin**：装完之后按一次 `Ctrl+R`，感受一下全文搜索和过滤功能，之后照常用，习惯会自然形成
- **zoxide**：装完之后照常 `cd` 进目录，积累几天记录之后试一次 `z 项目名`，就能体会到省了多少路径

系列前几篇装过的 fzf，配合 zoxide 的 `zi` 命令效果更好——这也是工具链组合带来的放大效应。
