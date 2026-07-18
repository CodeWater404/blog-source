---
title: "crontab 定时任务实战：时间字段、环境变量坑与日志排查"
date: 2026-07-17 17:41:40
updated: 2026-07-18 12:59:36
cover: /img/p53.jpg
categories: tools
tags:
  - Linux
  - crontab
  - 运维
  - CLI
description: "crontab 五个时间字段怎么写、-e/-l 常用操作、任务不执行九成是 PATH 问题：定时任务实战。"
---

脚本手动跑得好好的，放进 crontab 就是不执行——没有报错、没有输出，像被吞掉了一样。这是 crontab 最经典的翻车现场，九成指向同一个原因（后面讲），但先把 crontab 本身理顺。

<!-- more -->

这是 Linux 命令系列的一篇，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)。`cron` 是类 Unix 系统的定时任务守护进程，`crontab` 是管理它的命令——每个用户有一份自己的任务表，按设定的时间自动执行命令。

## 基本操作：-e、-l，以及要避开的 -r

```bash
crontab -e
#     编辑当前用户的任务表，用 $EDITOR 打开，保存退出即生效
#     每个用户一份独立的 crontab，互不干扰

crontab -l
#     列出当前用户的所有定时任务

crontab -r
#     ⚠️ 删除当前用户的整个任务表——没有任何确认提示，敲下回车全没了
#     -r 和 -e 在键盘上挨着，手滑的代价是全部任务瞬间蒸发
#     建议：永远用 -e 进去删掉某一行，别用 -r
```

被 `-r` 清空且没有备份的话，任务表找不回来。有个简单的保险：定期 `crontab -l > ~/crontab.bak` 备份一份。

## 五个时间字段：分 时 日 月 周

crontab 每行由五个时间字段加一条命令组成，字段顺序固定：

```text
┌───────── 分钟 (0-59)
│ ┌─────── 小时 (0-23)
│ │ ┌───── 日   (1-31)
│ │ │ ┌─── 月   (1-12)
│ │ │ │ ┌─ 周几 (0-7，0 和 7 都是周日)
│ │ │ │ │
*  *  *  *  *  要执行的命令
```

四种常用写法，组合起来能表达绝大多数调度需求：

```bash
0 3 * * * /opt/scripts/backup.sh
#     每天凌晨 3:00 执行——* 表示"每一个"，分钟填 0、小时填 3，其余全通配

*/5 * * * * /opt/scripts/check.sh
#     每 5 分钟执行一次——*/N 表示"每隔 N"

0 9 * * 1-5 /opt/scripts/report.sh
#     工作日（周一到周五）每天 9:00——N-M 表示连续范围

0 8 1,15 * * /opt/scripts/remind.sh
#     每月 1 号和 15 号的 8:00——N,M 表示离散的几个值
```

还有一组快捷写法，替代整个五字段：`@daily`（每天零点）、`@hourly`、`@weekly`、`@monthly`，以及比较特殊的 `@reboot`（系统启动时执行一次）。

## 任务不执行的头号原因：cron 的环境变量是极简的

回到开头的问题。cron 执行任务时**不加载你的 shell 配置**——`.bashrc`、`.zshrc`、`.profile` 统统不读，`PATH` 通常只有 `/usr/bin:/bin` 这么两截。你手动跑脚本没问题，是因为交互 shell 里 PATH 齐全；cron 里同一条命令找不到可执行文件，就静默失败了。

两种解法，任选或同时用：

```bash
which node
#     先用 which 查出命令的绝对路径，比如 /usr/local/bin/node

0 3 * * * /usr/local/bin/node /opt/app/task.js
#     解法一：crontab 里所有命令一律写绝对路径，包括脚本里调用的命令
```

```bash
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
0 3 * * * node /opt/app/task.js
#     解法二：在 crontab 文件顶部显式声明 PATH 和 SHELL
#     声明之后，下面所有任务行都在这个环境下执行
```

## 输出去哪了：重定向到日志

任务的输出（stdout 和 stderr）默认会尝试通过本地邮件发给用户——绝大多数服务器根本没配邮件服务，输出就这么无声消失了，出错也看不到错误信息。标准做法是自己重定向：

```bash
0 3 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
#     >> 追加写入日志文件（> 是覆盖写，日志要用追加）
#     2>&1 拆开看：程序有两条输出通道，1 是标准输出（正常输出），2 是标准错误（报错信息）
#     >> 只接管了通道 1；"2>&1" 的意思是"让通道 2 也指向通道 1 当前指向的地方"
#     合起来：正常输出和报错都进这个日志文件——排查任务失败时，报错信息才是最想要的那部分
#     顺序有讲究：2>&1 必须写在 >> 文件 的后面，先定好 1 去哪，2 才能跟对地方
```

怀疑任务压根没被触发（而不是执行出错）时，查 cron 自己的日志：

cron 服务的名字随发行版不同：Debian/Ubuntu 系叫 `cron`，RHEL/CentOS 系叫 `crond`——先确认自己机器上是哪个，再查它的日志：

```bash
systemctl list-units --type=service | grep -E "cron"
#     确认本机的服务名到底是 cron 还是 crond

journalctl -u cron -n 50
#     看 cron 服务最近 50 条日志，每次任务触发都有记录
#     RHEL/CentOS 系把 cron 换成 crond；journalctl 的过滤用法在 systemctl 那篇讲过
#     没有 systemd 的老系统翻 /var/log/syslog 或 /var/log/cron
```

日志里有触发记录但结果不对 → 执行环境问题（八成是上一节的 PATH）；连触发记录都没有 → 时间字段写错了。

## 一个专属坑：% 需要转义

crontab 的命令部分里，`%` 是特殊字符——未转义的 `%` 会被当成换行符，它之后的内容全部作为标准输入喂给命令（`man 5 crontab` 原文如此）。最常见的踩法是在文件名里用 `date` 生成日期：

```bash
0 3 * * * tar -czf /backup/logs-$(date +\%F).tar.gz -C /var/log . 
#     date +%F 输出 2026-07-17 这种格式，但 % 必须写成 \%
#     不转义的话命令会在 % 处被截断，任务安静地失败，非常难查
```

## 实战：每天凌晨增量备份

把前面的点串成一个真实任务——每天凌晨 3 点用 rsync 做增量备份，写日志、防 PATH 坑：

```bash
PATH=/usr/local/bin:/usr/bin:/bin
0 3 * * * rsync -a --delete /data/ /backup/data/ >> /var/log/backup.log 2>&1
#     rsync 的 -a、--delete 参数含义见 rsync 那篇
#     顶部声明 PATH，输出进日志，时间字段"每天 3:00"
```

需求再复杂一点——比如"上一次没跑完就别启动下一次""失败自动重试""依赖网络就绪"——crontab 就力不从心了，那是 systemd timer 的领域，配合 [systemctl 那篇](/systemctl-journalctl实战-服务管理与日志过滤) 的 unit 文件一起用。日常的定时脚本，crontab 足够。

## 常用操作速查表

| 操作/写法 | 作用 |
|---|---|
| `crontab -e` | 编辑任务表，保存即生效 |
| `crontab -l` | 查看任务表；`crontab -l > 备份文件` 定期备份 |
| `crontab -r` | 无确认清空全部任务，慎用，建议永远用 -e 删行 |
| 五字段顺序 | 分 时 日 月 周 |
| `*/5` / `1-5` / `1,15` | 每隔 5 / 连续范围 / 离散值 |
| `@daily` / `@reboot` | 每天零点 / 开机执行一次 |
| PATH 坑 | cron 不读 shell 配置，命令写绝对路径或顶部声明 PATH |
| `>> 日志 2>&1` | 输出重定向进日志，否则静默消失 |
| `journalctl -u cron` | 确认任务有没有被触发 |
| `\%` | 命令里的 % 必须转义，否则命令被截断 |

## 写到这里

crontab 的语法五分钟就能学会，真正拉开差距的是那几个坑：环境变量极简（绝对路径/声明 PATH）、输出会消失（重定向日志）、`%` 要转义、`-r` 别碰。把这四条焊在肌肉记忆里，定时任务"静默失败"的排查时间能从几小时缩到几分钟。
