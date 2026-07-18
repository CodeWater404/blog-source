---
title: "curl 与 wget 实战：调接口、下载文件与断点续传"
date: 2026-07-14 14:16:45
updated: 2026-07-18 16:44:36
cover: /img/p48.jpg
categories: tools
tags:
  - Linux
  - 网络
  - 命令行
  - CLI
description: "curl 调接口、wget 下载：-I/-L/-d 常用参数、断点续传与两个工具怎么分工。"
---

`curl` 和 `wget` 都能发 HTTP 请求、都能下载文件，为什么大多数系统里两个都有？因为它们的设计目标不同：`curl` 是数据传输的瑞士军刀，调接口、看响应头、在脚本里取数据都靠它；`wget` 是专职下载器，断点续传、批量下载是它的主场。分清分工，用起来就不纠结。

<!-- more -->

这是 Linux 命令系列的一篇，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)；[网络连通性诊断那篇](/网络连通性诊断-ping-traceroute与dig实战) 的排查流程里，确认网络和 DNS 都正常后验证"服务本身有没有响应"，用的正是 `curl`。

## curl 的默认行为：发 GET，结果打到屏幕

`curl` 不带任何参数时发 GET 请求，把响应体原样打印到标准输出——不落盘。这个默认行为正是它适合调试的原因：请求发出去，响应立刻看得见，还能直接接管道处理：

```bash
curl https://api.example.com/health
#     发 GET 请求，响应体直接打印到终端

curl -s https://api.example.com/users | jq '.[0].name'
#     -s：silent，去掉进度条等杂音输出，适合接管道
#     配合 jq 直接从 JSON 响应里取字段
```

## curl 调接口：高频参数

```bash
curl -I https://example.com
#     -I：只请求并显示响应头（实际发的是 HEAD 请求，不下载响应体）
#     看状态码、Content-Type、缓存头，不关心内容时用它最快

curl -v https://api.example.com/user
#     -v：verbose，显示完整的请求和响应过程——包括 DNS 解析、TLS 握手、
#     发出去的请求头、收到的响应头，调试"请求到底发成什么样了"必用

curl -X POST https://api.example.com/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123"}'
#     -X：指定请求方法；-H：加请求头；-d：请求体数据
#     其实 -d 本身就会把方法切成 POST，这里的 -X POST 可以省略，写出来更直观

curl -L http://example.com
#     -L：跟随 301/302 重定向——curl 默认不跟随，页面搬家了只会给你一个 301 响应

curl --max-time 5 https://api.example.com/health
#     --max-time：整个请求的超时上限（秒），脚本里调接口一定要加，避免卡死

curl -s -o /dev/null -w "%{http_code}\n" https://example.com
#     -w：请求完成后按模板输出元信息，%{http_code} 是状态码
#     -o /dev/null 丢弃响应体——组合起来就是"只要状态码"，健康检查脚本的标准写法
```

## curl 下载文件

```bash
curl -o app.tar.gz https://example.com/download/app-v2.tar.gz
#     -o：把响应体保存成指定文件名，代替默认的打印到终端

curl -O https://example.com/download/app-v2.tar.gz
#     -O：直接用 URL 里的文件名保存（这里存成 app-v2.tar.gz）
```

偶尔下载一两个文件，`curl -O` 够用；但要断点续传、批量下载，就该换 `wget` 了。

## curl -sSL | sh：装软件时最常见的组合

很多开源工具的官方安装方式都是这个模式——curl 拉下安装脚本，直接通过管道交给 shell 执行：

```bash
curl -sSL https://example.com/install.sh | sh
#     -s：silent，关掉进度条和统计信息——脚本内容要进管道，进度条会污染输出
#     -S：show-error，配合 -s 使用——静默模式下如果请求失败，仍然把错误信息打出来
#         只用 -s 的话连报错都吞掉，下载失败了管道那头收到空内容，莫名其妙
#     -L：跟随重定向——GitHub raw、安装脚本短链这类地址几乎都有跳转，不加拿到的是空响应
#     | sh：把下载到的脚本内容直接交给 shell 执行，一步完成"下载+安装"
```

`-sSL` 三个参数几乎总是绑在一起出现，可以当成固定搭配记。另外提醒一句：这个模式等于"把远程内容直接交给 shell 执行"，只对信任的官方源用；对来路不明的脚本，先把 `| sh` 去掉，把内容下载下来看一眼再决定跑不跑。

## wget 的默认行为：直接存成文件

`wget` 和 `curl` 的默认行为正相反：不带参数就把 URL 内容保存成本地文件（文件名取自 URL），从设计上就是给"下载"这个场景准备的。macOS 自带 `curl` 但不自带 `wget`，需要 `brew install wget`：

```bash
wget https://example.com/download/app-v2.tar.gz
#     直接下载保存为 app-v2.tar.gz，带进度条和速度显示

wget -c https://example.com/download/big-file.iso
#     -c：continue，断点续传——下载中断后重跑同一条命令，从断的地方继续
#     下载几个 G 的大文件时，这是 wget 相对 curl 最实用的优势

wget -P /tmp/downloads https://example.com/file.zip
#     -P：指定保存目录，不加就存当前目录

wget -r --no-parent https://example.com/docs/
#     -r：递归下载，把页面里链接的资源一并抓下来
#     --no-parent：不往上层目录爬，把范围限制在 /docs/ 之下
#     递归下载对目标网站压力不小，范围一定要限制好，别对着整站开抓
```

## 怎么选：一句话分工

- 调接口、看响应头、脚本里取数据、健康检查——用 `curl`，默认输出到 stdout 天生适合这些场景
- 下载大文件要断点续传、批量抓取——用 `wget`，`-c` 和 `-r` 是 `curl` 没有的便利
- 只是偶尔下载个小文件，两者都行，`curl -O` 和 `wget` 等价，用系统里现成的那个（macOS 现成的是 `curl`）

## 常用参数速查表

| 命令/参数 | 作用 |
|---|---|
| `curl URL` | 发 GET，响应体打印到终端 |
| `curl -I` | 只看响应头（HEAD 请求） |
| `curl -v` | 显示完整请求/响应过程，调试用 |
| `curl -X 方法` | 指定请求方法（POST/PUT/DELETE 等） |
| `curl -H "头: 值"` | 添加请求头，可写多次 |
| `curl -d '数据'` | 发送请求体，同时自动把方法切成 POST |
| `curl -L` | 跟随重定向，默认不跟随 |
| `curl --max-time N` | 整个请求的超时上限（秒） |
| `curl -o 文件` | 响应体保存为指定文件名 |
| `curl -O` | 响应体保存为远端 URL 里的文件名 |
| `curl -s` | 静默模式，关掉进度条和统计信息 |
| `curl -S` | 配合 `-s`：静默但请求失败时仍显示错误 |
| `curl -sSL url \| sh` | 安装脚本固定搭配：以上三个参数连用 + 管道执行 |
| `curl -w "%{http_code}"` | 输出状态码等元信息 |
| `wget URL` | 下载保存为文件（默认行为） |
| `wget -c` | 断点续传 |
| `wget -P 目录` | 指定保存目录 |
| `wget -r` | 递归下载，把页面里链接的资源一并抓下来 |
| `wget --no-parent` | 配合 `-r`：不向上层目录爬，限制递归范围 |

## 写到这里

记住两个默认行为就分清了：`curl` 默认打到屏幕（所以是调试工具），`wget` 默认存成文件（所以是下载器）。调接口的高频参数就 `-I`/`-v`/`-H`/`-d`/`-L` 这几个，下载大文件认准 `wget -c`，其余参数用到时查表即可。
