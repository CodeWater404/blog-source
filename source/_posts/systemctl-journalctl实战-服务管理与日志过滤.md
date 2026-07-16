---
title: "systemctl 与 journalctl 实战：enable 和 start 的区别、unit 文件与日志过滤"
date: 2026-07-14 20:46:22
cover: /img/p50.jpg
categories: tools
tags:
  - Linux
  - systemd
  - 运维
  - CLI
description: "systemctl 的 enable 和 start 有什么区别、unit 文件放哪、journalctl 怎么按时间和级别过滤日志。"
---

服务器重启之后服务没起来，明明之前 `systemctl start` 跑得好好的——因为 `start` 和 `enable` 是两件独立的事，只做了前者，服务就只活到下一次重启为止。这类概念没对齐的坑，systemd 里还有几个。

<!-- more -->

这是 Linux 命令系列的一篇，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)；[服务器排查那篇](/服务器排查服务问题常用命令) 列了 systemctl/journalctl 的基本命令，这篇讲背后的概念和进阶用法。

## systemd 和 unit：先对齐两个词

systemd 是现代主流 Linux 发行版（Ubuntu 16.04+、CentOS 7+、Debian 8+）的服务管理器，负责开机按依赖顺序拉起服务、运行期间盯着它们的状态。它管理的最小单位叫 unit——服务只是其中最常见的一类（`.service`），此外还有定时器（`.timer`）、挂载点（`.mount`）等类型。日常说"管服务"，操作的就是 service 类型的 unit。

## start 管现在，enable 管开机：两件独立的事

```bash
systemctl start nginx
#     现在就把服务跑起来——但只管这一次，重启机器后不会自动再起

systemctl enable nginx
#     设置开机自启——但不影响现在，跑这条命令时服务并不会立刻启动

systemctl enable --now nginx
#     两件事一步做完：设置开机自启，并且现在就启动

systemctl disable nginx
#     取消开机自启，同样不影响当前运行状态

systemctl is-enabled nginx
#     查询是否已设置开机自启，输出 enabled / disabled
```

两者完全正交：`start` 了没 `enable`，重启后服务消失；`enable` 了没 `start`，当前这会儿服务并没有跑。部署新服务的标准动作是 `enable --now`，一次把两头都管上。

`stop` 的行为也值得知道：它默认先给服务进程发 SIGTERM 请求优雅退出，超时还没退才升级为 SIGKILL——正是 [kill 信号那篇](/kill信号机制与nice优先级实战) 讲的"先礼后兵"流程，systemd 帮你做好了。

## systemctl status 输出怎么读

```text
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; ...)
     Active: active (running) since Tue 2026-07-14 09:00:12 CST; 2h ago
   Main PID: 1235 (nginx)
     Memory: 12.4M
#     Loaded 行看两个信息：unit 文件在哪、有没有设置开机自启（enabled/disabled）
#     Active 行是当前状态：active (running) 正常；failed 是启动失败——
#     failed 时下面通常直接附着最近几条报错日志，排查从这里入手最快
```

## unit 文件放哪、怎么改

unit 文件有两个主要目录，优先级不同：

- `/lib/systemd/system/`（部分发行版是 `/usr/lib/systemd/system/`）：包管理器安装软件时放的，**别直接改**——包升级会把改动覆盖掉
- `/etc/systemd/system/`：管理员自己的地盘，优先级更高——同名 unit 在两边都存在时，以这里的为准

自己部署一个 Go 服务，最小可用的 unit 文件长这样：

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Go App
After=network.target
#     After：等网络就绪后再启动本服务

[Service]
ExecStart=/opt/myapp/myapp
#     要运行的命令，必须是绝对路径
Restart=on-failure
#     进程异常退出时自动拉起，正常 stop 不会触发
User=myapp
#     用哪个用户身份运行，不写默认 root——生产环境建议专门建个低权限用户

[Install]
WantedBy=multi-user.target
#     enable 时挂到哪个启动目标下，服务类 unit 基本都写这个
```

改完 unit 文件（新建、修改都算）有一步必做：

```bash
systemctl daemon-reload
#     让 systemd 重新读取 unit 文件——不跑这条，systemd 继续用内存里的旧配置，
#     你改的内容看起来"完全不生效"，这是 unit 文件改了没反应的头号原因

systemctl enable --now myapp
#     然后正常启动
```

## journalctl：按服务、时间、级别过滤日志

systemd 托管的服务，标准输出和标准错误都被 journald 收集，用 `journalctl` 查，不用满盘找日志文件：

```bash
journalctl -u myapp
#     -u：只看指定服务的日志，从最早开始分页显示

journalctl -u myapp -f
#     -f：实时跟踪新日志，等价于 tail -f 的体验

journalctl -u myapp -n 100
#     -n：只看最近 100 行

journalctl -u myapp --since "2026-07-14 09:00" --until "2026-07-14 10:00"
#     --since/--until：按时间段过滤，也接受 "1 hour ago"、"yesterday" 这类相对写法

journalctl -u myapp -p err
#     -p：按级别过滤，err 只看错误及以上（emerg/alert/crit/err）

journalctl -u myapp -b
#     -b：只看本次开机以来的日志——排查"重启后起不来"时配合 -p err 特别好用
```

## 常用命令速查表

| 命令 | 作用 |
|---|---|
| `systemctl start/stop/restart 服务` | 启动 / 停止 / 重启（只管当下） |
| `systemctl reload 服务` | 平滑重载配置，不中断服务（需服务支持） |
| `systemctl enable --now 服务` | 设置开机自启并立刻启动 |
| `systemctl is-enabled 服务` | 查是否开机自启 |
| `systemctl status 服务` | 状态 + 最近日志，看 Loaded/Active 两行 |
| `systemctl daemon-reload` | 改过 unit 文件后必跑，否则改动不生效 |
| `journalctl -u 服务 -f` | 实时跟踪服务日志 |
| `journalctl -u 服务 --since "1 hour ago"` | 按时间过滤 |
| `journalctl -u 服务 -p err -b` | 本次开机以来的错误日志 |

## 写到这里

systemd 的坑基本都是概念错位：`start` 和 `enable` 各管一头（记住 `enable --now`）、unit 文件改在 `/etc/systemd/system/` 且改完必须 `daemon-reload`。日志这边 `journalctl -u` 加上时间和级别过滤，比去 `/var/log` 翻文件体面得多。这套流程走顺了，自己部署的服务和发行版自带的服务就是同一套管理方式。
