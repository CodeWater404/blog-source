---
title: "mise：一个工具管住所有语言版本，不用再装一堆 nvm/gvm"
date: 2026-08-22 00:33:11
categories: tools
tags:
  - mise
  - 版本管理
  - 开发效率
cover: /img/p67.jpg
description: "mise 是什么、怎么配置、怎么用——一个工具替代 nvm/gvm 这类单语言版本管理器，附常见语言的完整配置示例。"
---

`nvm` 管 Node 版本，`gvm` 管 Go 版本，再遇上 Python 项目大概率还得装个 `pyenv`——每种语言一个版本管理器，每个都要单独学一套命令、单独在 shell 配置文件里加一段初始化脚本。`mise` 要解决的就是这个问题：一个工具，管所有语言的版本，命令和配置格式都是统一的一套。

<!-- more -->

## mise 是什么

`mise` 是一个通用的开发环境管理工具，核心能力是按项目目录管理不同语言/工具的版本——进到不同的项目目录，`mise` 会自动切换成这个项目声明的 Node、Go、Python 版本，不用手动 `nvm use` 或者 `gvm use` 切来切去。除此之外它还顺带管环境变量、管项目任务（跑测试、跑构建这类命令），但版本管理是最核心、最先能用上的部分。

## 安装

macOS 和 Linux 上官方推荐用它自己的安装脚本，装的是编译优化过的官方二进制，性能和更新速度都比包管理器装的版本更好：

```bash
curl https://mise.run | sh
#     装到 ~/.local/bin/mise，装完不需要手动加进 PATH——
#     只要在 shell 配置文件里加了下面的 activate 那一行，mise 会自己把自己加进 PATH
```

如果更习惯用 Homebrew，也能装，只是官方原话说明这不是首选方式（Homebrew 编译的版本不如官方二进制优化得好，更新也可能滞后于官方发布）：

```bash
brew install mise
```

装完之后，把下面这行加进 shell 的配置文件（`~/.zshrc`、`~/.bashrc`，看你用什么 shell）：

```bash
eval "$(mise activate zsh)"
#     zsh 换成你自己用的 shell，比如 bash / fish
```

重启一下终端（或者 `source` 一下配置文件），跑 `mise doctor` 确认一切正常。装好之后最简单的用法：

```bash
mise use --global node@22
#     全局默认用 Node 22

mise use go@1.25
#     只在当前项目目录用 Go 1.25，写进当前目录的 mise.toml
```

`mise use` 不带 `--global` 时，会在当前目录生成或更新一份 `mise.toml`，把这个版本要求记录下来——这份文件提交进项目仓库，队友 `clone` 下来之后跑一次 `mise install`，版本就跟你保持一致，不用每个人各自手动装。

## 跟 nvm、gvm 比，区别在哪

`nvm`、`gvm` 都是只管一种语言的版本管理器，而且都是靠 shell 脚本实现的：装好之后要在 `~/.bashrc` 或者 `~/.zshrc` 里手动加一行 `source` 脚本的命令，每次开新终端都要重新加载这段脚本，`nvm use`、`gvm use` 也只在当前这个 shell 会话里生效。项目里同时用了 Node、Go、Python，就要把 `nvm`、`gvm`、`pyenv` 三样都装上、三套命令都记住、三份类似 `.nvmrc`/`.go-version`/`.python-version` 的文件都维护着。

`mise` 是一个用 Rust 写的单一二进制程序，不是靠 shell 脚本拼起来的——前面加的那行 `eval "$(mise activate zsh)"` 装好之后就会在每次 prompt 刷新的时候自动检查当前目录该用哪个版本，自动把对应的 `bin` 目录放到 `PATH` 最前面，不用像 `nvm use`/`gvm use` 那样每次手动敲一遍。所有语言共用同一份 `mise.toml`、同一套 `mise use`/`mise install` 命令，学一次就够了。

## 配置文件长什么样

一个覆盖了常见几种语言的 `mise.toml`：

```toml
[tools]
node = "22"
go = "1.25"
python = "3.13"
ruby = "3.4"
rust = "1.83"
java = "21"

[env]
DATABASE_URL = "postgres://localhost/myapp_dev"

[tasks.build]
run = "go build -o bin/app ."

[tasks.test]
run = "go test ./..."
```

`[tools]` 里每一行就是"这个语言用这个版本"，常见语言 `mise` 都内置了支持，不用额外装插件。`[env]` 是这个项目目录下自动生效的环境变量，进这个目录就有、出了这个目录就没有。`[tasks]` 定义的命令用 `mise run build` 或者简写 `mise build` 执行——这部分不是必需的，只用版本管理这一个功能也完全没问题，配置文件里只写 `[tools]` 也是合法的。

## 用之前要知道的一件事：配置文件能跑代码，需要手动信任

`mise.toml` 里的 `[env]`、任务这些不是静态声明，是真的会执行的——如果你 `clone` 了一个别人写的项目，第一次在这个目录里用 `mise`，它会提示你先手动确认信任这份配置文件：

```bash
mise trust
#     信任当前目录下的配置文件，做一次就够
```

自己用 `mise use` 生成的配置会被自动信任，只有"拉取别人写的仓库、第一次遇到里面的 mise.toml"这种情况才会跳出这个提示。看到提示的时候，打开配置文件扫一眼 `[env]` 和任务里写了什么，比直接无脑点信任更稳妥。

## 用之前要知道的另一件事：找不到版本时不一定会报错

如果项目里用的是 `mise activate --shims` 这种模式（在 IDE、CI 这类非交互式环境里比较常见），遇到某个版本还没装好的情况，`mise` 不会直接报错，而是会**悄悄换成系统上能找到的第一个同名程序**顶替上去。比如 Linux 发行版大多自带一个系统级的 `python3`，如果 `mise` 要的版本没装好，很可能就默默用了这个系统自带的 `python3`，而不是提示"你要的版本没找到"——脚本照样能跑，只是跑在了一个你没料到的解释器上，出问题也很难第一时间想到是版本不对。想让这种情况直接报错，需要把 `not_found_auto_install` 和 `not_found_system_fallback` 这两个配置项都关掉。
