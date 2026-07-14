---
title: "apt、yum/dnf、brew 包管理实战：换发行版不用重新查命令"
date: 2026-07-14 10:07:32
cover: /img/p43.jpg
categories: tools
tags:
  - Linux
  - 包管理
  - 命令行
  - CLI
description: "apt、yum/dnf、brew 常用操作怎么对应：安装、卸载、搜索、源配置一次讲清楚。"
---

本地 Mac 用 `brew`，公司服务器是 Ubuntu 用 `apt`，偶尔还要连一台 CentOS 用 `yum`——同样是"装个包"，三套命令语法完全不一样，来回切换很容易记混，装个包先愣一下"这里到底该敲哪个"。

<!-- more -->

这是 Linux 命令系列的一篇，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)。

## 三者分别是谁

`apt`（Advanced Package Tool）是 Debian、Ubuntu 系发行版的包管理器，2016 年 Ubuntu 16.04 引入，把之前 `apt-get`/`apt-cache` 最常用的功能整合成一个更友好的命令，默认还带进度条和彩色输出。

`yum`/`dnf` 是 RHEL、CentOS、Fedora 系发行版的包管理器。`dnf` 是 `yum` 的继任者，Fedora 从第 22 版（2015 年）起默认用 `dnf`，RHEL 8 / CentOS 8（2019 年）起也切换为默认用 `dnf`——现在的系统上敲 `yum`，背后实际调用的通常就是 `dnf`。

`brew`（Homebrew）是 macOS（以及部分 Linux 发行版）的包管理器，装在用户目录下，不需要 `sudo`。

## 常用操作对照

装包、卸载、搜索这几个高频操作，三套命令放在一起看最清楚：

```bash
# 更新软件源信息（不是升级软件本身，只是刷新"有哪些包、什么版本"这份清单）
apt update
dnf check-update
brew update

# 安装一个包
apt install nginx
dnf install nginx
brew install nginx

# 卸载一个包
apt remove nginx
dnf remove nginx
brew uninstall nginx

# 搜索一个包
apt search nginx
dnf search nginx
brew search nginx

# 列出已安装的包
apt list --installed
dnf list installed
brew list

# 查看某个包的详细信息
apt show nginx
dnf info nginx
brew info nginx
```

三者语义基本一一对应，记住"动作"（装/卸/搜/查）而不是死记每个发行版的具体单词，切换起来会顺手很多。

## 升级：更新源信息和升级已装软件是两回事

`update` 类命令容易被误解成"升级软件"，实际上它只刷新本地的软件源索引（有哪些包、哪个版本），真正把已装软件升级到新版本要用另一条命令：

```bash
# Debian/Ubuntu：先刷新索引，再升级已装的包
apt update && apt upgrade

# RHEL/CentOS/Fedora：dnf 的 upgrade 会自动先检查更新
dnf upgrade

# macOS：brew upgrade 升级所有已装的包（不加包名）
brew update && brew upgrade
```

## 仓库和权限模型的本质区别

`apt` 和 `dnf` 依赖系统级的仓库配置文件——`apt` 是 `/etc/apt/sources.list`（以及 `/etc/apt/sources.list.d/` 目录下的分文件配置），`dnf` 是 `/etc/yum.repos.d/` 下的一堆 `.repo` 文件。安装软件要写入系统目录（`/usr/bin`、`/etc` 等），所以需要 `sudo` 权限。

`brew` 走的是完全不同的模型：软件仓库叫 tap，包和依赖全部装在用户目录下——Apple Silicon Mac 上是 `/opt/homebrew`，Intel Mac 上是 `/usr/local`（本机实测 `brew --prefix` 输出 `/opt/homebrew`）。因为不碰系统目录，`brew` 全程不需要 `sudo`，这也是它能在 macOS 这种对系统目录管得很严的系统上顺畅工作的原因。

## yum 还是 dnf：直接用 dnf

新装的 RHEL 8 / CentOS 8 及更新版本上，`yum` 命令依然能用，但只是指向 `dnf` 的兼容别名，背后真正执行的是 `dnf`。日常直接敲 `dnf` 就好，看到别人的脚本或者旧文档里写 `yum`，心里知道它现在等价于 `dnf` 即可，不用纠结该学哪个。

## 三大家族命令速查表

| 操作 | apt（Debian/Ubuntu） | dnf（RHEL/CentOS/Fedora） | brew（macOS） |
|---|---|---|---|
| 刷新源索引 | `apt update` | `dnf check-update` | `brew update` |
| 安装包 | `apt install pkg` | `dnf install pkg` | `brew install pkg` |
| 卸载包 | `apt remove pkg` | `dnf remove pkg` | `brew uninstall pkg` |
| 搜索包 | `apt search pkg` | `dnf search pkg` | `brew search pkg` |
| 列出已装 | `apt list --installed` | `dnf list installed` | `brew list` |
| 查看包信息 | `apt show pkg` | `dnf info pkg` | `brew info pkg` |
| 升级已装软件 | `apt upgrade` | `dnf upgrade` | `brew upgrade` |
| 是否需要 sudo | 需要 | 需要 | 不需要 |

## 写到这里

三套包管理命令的核心操作就这几个：装、卸、搜、查、升级，语义上是对齐的，差的只是具体单词和权限模型。换发行版或者换到 Mac，不用重新学一遍，对照着上面这份表格套用命令名就行。
