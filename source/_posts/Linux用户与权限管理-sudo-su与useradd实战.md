---
title: "Linux 用户与权限管理：sudo、su 与 useradd 实战"
date: 2026-07-14 10:09:44
cover: /img/p44.jpg
categories: tools
tags:
  - Linux
  - 权限
  - 命令行
  - CLI
description: "sudo 和 su 区别在哪，useradd 建用户为什么没有家目录：常见权限操作实战。"
---

需要用另一个身份执行操作时，`sudo`、`su`、`sudo su -` 好几种写法混着搜到的答案都不一样——都能"变成别的用户"，但具体差在哪、该用哪个，很少有人说清楚。

<!-- more -->

这是 Linux 命令系列的一篇，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)；跟 [Linux 权限与属主管理：chmod、chown](/Linux权限与属主管理-chmod-chown实战) 是互补关系——那篇讲文件权限本身，这篇讲"谁能以什么身份操作"。

## sudo：临时提权执行单条命令

`sudo`（superuser do）默认行为是以另一个用户（不指定的话是 root）的身份执行紧跟在后面的这一条命令，执行完就恢复回当前用户——不是切换整个会话。使用前必须在 `/etc/sudoers` 里被授权（通常是加入 `sudo` 或 `wheel` 组），没有授权直接报错拒绝。

```bash
sudo systemctl restart nginx
#     以 root 身份重启 nginx，执行完这条命令，当前 shell 还是原来的用户身份

sudo -u www-data whoami
#     -u：指定以哪个用户执行，不写默认是 root
```

需要连续跑好几条 root 命令时，有两种进入"root 状态"的写法，行为不一样：

```bash
sudo -i
#     完整登录 shell，加载 root 自己的环境变量和 profile，工作目录切到 /root
#     -i 是 --login 的简写

sudo -s
#     保留当前用户的环境变量和 shell 配置，只是提权，不会进入 root 的家目录
```

`sudo -i` 更"干净"——环境完全是 root 自己的，不会被当前用户 shell 里可能存在的异常配置影响；`sudo -s` 适合只是想临时用 root 权限跑几条命令、又不想离开当前 shell 环境的场景。

## su：切换到另一个用户的完整会话

`su`（switch user）是切换到另一个用户的整个 shell 会话，不像 `sudo` 那样执行完一条命令就自动恢复。同样有一个容易被忽略的区别：

```bash
su alice
#     切换到 alice，但保留当前的环境变量和所在目录（$HOME 还是原来的）

su - alice
#     加了 -（等价 su -l alice）：完整登录，环境变量、$HOME、当前工作目录
#     全部换成 alice 自己的，就像重新登录了一次
```

不加 `-` 容易踩坑：切换过去之后 `$HOME`、`$PATH` 这些变量其实还是原来用户的，跑一些依赖环境变量的脚本可能出现莫名其妙的行为——日常切换身份，`su -` 才是更安全的默认选择。

## sudo 和 su 该怎么选

- 只需要执行单条命令，用 `sudo`——每次操作都过一遍 sudoers 授权检查，操作有日志可查（记录谁在什么时间执行了什么），更适合团队协作的服务器环境
- 需要以另一个身份连续操作一段时间，用 `su -` 或者 `sudo su -`（先用 sudo 权限切换到 su，再完整登录）
- 现代实践更推荐前者：只在需要的时候临时提权，用完就退出，比长时间挂在一个 root 会话里风险更小

## useradd / usermod：创建和修改用户

```bash
useradd -m -s /bin/bash -G sudo alice
#     -m：创建家目录——不加这个参数，且系统的 CREATE_HOME 没打开的话，
#         默认不会创建家目录，这是新建用户后"登录就报错找不到家目录"的常见原因
#     -s：指定登录 shell，这里是 bash
#     -G：加入附加组，这里是把 alice 加进 sudo 组，让她能使用 sudo 命令
```

修改已有用户的这些属性，用 `usermod`，参数含义基本和 `useradd` 一致：

```bash
usermod -aG docker alice
#     -aG：把 alice 追加（append）进 docker 组
#     注意一定要带 -a，只写 -G 会用新列表整个替换掉原来的附加组，把之前加入的组全部移除

usermod -s /bin/zsh alice
#     修改 alice 的默认登录 shell
```

## id / whoami：查看当前身份

```bash
id
#     显示当前用户的 UID、GID，以及所属的所有组

whoami
#     只输出当前用户名，不显示 UID/GID 和组信息
```

`id` 信息更全，排查"这个用户到底在哪些组里、有没有权限做某件事"时用它；只是想确认"我现在是谁"，`whoami` 更快。

## 常用命令速查表

| 命令 | 作用 |
|---|---|
| `sudo 命令` | 以另一个用户（默认 root）身份执行单条命令，执行完恢复原身份 |
| `sudo -i` | 完整登录 shell，加载 root 自己的环境变量，进入 /root |
| `sudo -s` | 保留当前用户的环境变量，只提权 |
| `su 用户名` | 切换用户但保留当前环境变量和目录 |
| `su - 用户名` | 完整登录，环境变量、$HOME、当前目录全部换成目标用户的 |
| `useradd -m` | 创建用户并生成家目录 |
| `useradd -s` | 指定登录 shell |
| `useradd -G` | 加入附加组 |
| `usermod -aG` | 追加加入某个组，不会移除已有的附加组 |
| `id` | 查看当前用户的 UID/GID 和所属组 |
| `whoami` | 只查看当前用户名 |

## 写到这里

`sudo` 和 `su` 的核心区别是"临时提权一条命令"还是"切换整个会话"；`su`/`su -` 的核心区别是环境变量要不要一起换。记住这两组对比，剩下的 `useradd`/`usermod`/`id`/`whoami` 都是围绕"这个用户是谁、能做什么"展开的具体操作，日常运维基本够用。
