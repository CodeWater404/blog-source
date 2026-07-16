---
title: "SSH 免密登录与 scp 实战：密钥、config 别名与传文件"
date: 2026-07-14 20:48:27
updated: 2026-07-16 16:32:14
cover: /img/p51.jpg
categories: tools
tags:
  - Linux
  - SSH
  - 命令行
  - CLI
description: "ssh-keygen 生成密钥、ssh-copy-id 免密登录、config 起别名、scp 传文件（-P 是大写）一篇配齐。"
---

每次连服务器输一遍密码，IP 和端口记在备忘录里翻着敲——这套体验其实配置一次就能永久甩掉：密钥登录免掉密码，config 别名免掉记 IP，剩下的就是 `ssh myserver` 三个词连上任何机器。

<!-- more -->

这是 Linux 命令系列的一篇，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)。

## ssh 基本连接

`ssh` 是加密的远程登录协议和客户端命令，连上之后拿到的就是远程机器的 shell：

```bash
ssh deploy@192.168.1.100
#     以 deploy 用户身份连接目标主机，默认端口 22

ssh -p 2222 deploy@192.168.1.100
#     -p：指定端口（小写 p）——注意这个细节，后面讲 scp 时它会变成一个坑
```

## ssh-keygen：生成密钥对

密钥登录的原理一句话：生成一对钥匙，**公钥放到服务器上，私钥留在本地绝不外传**；连接时客户端用私钥证明身份，服务器用公钥验证，全程不需要密码。

```bash
ssh-keygen -t ed25519 -C "you@example.com"
#     -t：密钥类型，ed25519 是当前 OpenSSH 的默认推荐——密钥短、速度快、安全性好
#     -C：注释，通常写邮箱，方便在服务器上区分这是谁的哪把钥匙
#     一路回车用默认路径（~/.ssh/id_ed25519），会问要不要设 passphrase：
#     设了更安全（私钥文件被偷也用不了），个人机器图省事可以留空
```

生成后 `~/.ssh/` 下多出两个文件：`id_ed25519` 是私钥（权限 600，谁都别给），`id_ed25519.pub` 是公钥（就是要放到服务器上的那个）。

需要管理多台机器、每台用独立密钥时（比如公司服务器一把、自己 VPS 一把、GitHub 再一把），用 `-f` 给每把钥匙指定自己的文件名，不然后生成的会覆盖先生成的：

```bash
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519_work
#     -f：指定密钥文件路径和名称，生成 id_ed25519_work（私钥）和
#     id_ed25519_work.pub（公钥），不动默认的 id_ed25519

ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519_vps
#     再给自己的 VPS 单独生成一把，两把钥匙互不干扰
```

多把钥匙怎么让 ssh 知道连哪台机器用哪把？靠的就是后面 `~/.ssh/config` 里的 `IdentityFile` 字段——每台服务器的配置段里指定自己的那把私钥。

## ssh-copy-id：把公钥装到服务器上

```bash
ssh-copy-id deploy@192.168.1.100
#     把本地公钥追加到服务器的 ~/.ssh/authorized_keys 里
#     这一步还需要输一次密码——这是最后一次，装好之后就免密了

ssh deploy@192.168.1.100
#     再连就直接进去了，不再询问密码
```

没有 `ssh-copy-id` 的环境（比如从 Windows 连），手动做等价的事：

```bash
cat ~/.ssh/id_ed25519.pub | ssh deploy@192.168.1.100 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
#     把公钥内容追加到服务器的 authorized_keys 文件
```

免密不生效时，第一个该查的是服务器端的目录权限——sshd 对权限要求很严格，太宽松会直接拒绝密钥登录：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
#     ~/.ssh 必须 700、authorized_keys 必须 600（或更严）
#     权限位的含义见 chmod 那篇：700 = 只有属主能读写进入
```

## ~/.ssh/config：给服务器起别名

密钥解决了密码，config 解决记 IP。在本地 `~/.ssh/config` 里给每台服务器写一段：

```text
Host myserver
    HostName 192.168.1.100
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
#     Host：你给它起的别名，后面连接时用这个名字
#     HostName：真实 IP 或域名
#     User/Port：默认用户和端口，连接时就不用每次写
#     IdentityFile：指定用哪把私钥，只有一把钥匙时可以省略
```

配好之后：

```bash
ssh myserver
#     等价于 ssh -p 2222 -i ~/.ssh/id_ed25519 deploy@192.168.1.100
```

管理多台服务器时价值更明显——每台一段配置，名字起得有意义（`prod-web`、`test-db`），连哪台敲哪个名字，IP 和端口再也不用记。`scp`、`rsync` 也认这份配置，别名全程通用。

## scp：基于 SSH 传文件

`scp` 走 SSH 通道在本地和远程之间拷贝文件，语法和 `cp` 一致——把其中一端写成 `用户@主机:路径` 就行：

```bash
scp app.tar.gz myserver:/opt/deploy/
#     上传：本地文件 → 远程目录（这里直接用了 config 里的别名）

scp myserver:/var/log/app.log ./
#     下载：远程文件 → 本地当前目录

scp -r ./dist myserver:/opt/deploy/
#     -r：整个目录递归拷贝
```

一个必踩一次的坑：**scp 指定端口用大写 -P**，和 ssh 的小写 `-p` 不一样——因为 scp 的小写 `-p` 已经被"保留文件的修改时间和权限位"占用了（man 手册原文就是这么解释的）：

```bash
scp -P 2222 app.tar.gz deploy@192.168.1.100:/opt/deploy/
#     -P：端口，大写——写成小写 -p 不会报错，但那是"保留时间戳"的意思，
#     端口会走默认 22，连接失败还不容易看出原因
#     用 ~/.ssh/config 配好别名可以彻底绕开这个坑：端口写在配置里
```

偶尔传一两个文件 `scp` 最顺手；大目录、大文件、需要断点续传或增量同步的场景，换 [rsync](/rsync增量同步实战-常用参数与远程同步) 更合适。

## 常用命令速查表

| 命令 | 作用 |
|---|---|
| `ssh user@host` | 远程登录，默认端口 22 |
| `ssh -p 端口` | 指定端口（小写 p） |
| `ssh-keygen -t ed25519` | 生成密钥对，ed25519 是当前默认推荐 |
| `ssh-copy-id user@host` | 把公钥装到服务器，装完免密 |
| `chmod 700 ~/.ssh` + `600 authorized_keys` | 免密不生效先查这个权限 |
| `~/.ssh/config` | 别名配置，`ssh 别名` 直连，scp/rsync 通用 |
| `scp 文件 host:路径` | 上传；反过来写就是下载 |
| `scp -r` | 整目录递归拷贝 |
| `scp -P 端口` | 指定端口是大写 P，小写 p 是保留时间戳 |

## 写到这里

一次性的配置换永久的顺手：`ssh-keygen` 生成 ed25519 密钥、`ssh-copy-id` 装到服务器、`~/.ssh/config` 起好别名，之后连任何机器都是 `ssh 名字` 一步到位。传文件 `scp` 顺手用，记住端口是大写 `-P`；要增量和断点续传，隔壁 `rsync` 伺候。
