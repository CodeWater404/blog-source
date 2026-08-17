---
title: "Go 热重载：air 怎么工作的，以及它和优雅关闭的一个冲突"
date: 2026-07-31 19:31:37
categories: golang
tags:
  - Go
  - 热重载
  - 开发效率
cover: /img/p65.jpg
description: "Go 里的热重载不是真的热更新，是监听变化重新编译再重启进程。air 默认还会直接强杀旧进程，跟优雅关闭逻辑冲突。"
---

先澄清一个容易想岔的地方："热重载"这个词在 Node.js、Python 那些解释型语言的语境里，经常指真的不重启进程、只替换某个模块的实现；但在 Go 的开发工具（`air`、`CompileDaemon`、`reflex` 这些）语境里，"热重载"从来都不是这个意思。air 官方 README 里原话就写着这句：这个工具跟生产环境的热部署没有任何关系。它做的事情始终是同一套：监听文件变化、重新跑一遍 `go build`、杀掉旧进程、启动编译出来的新二进制——是重启，不是热更新。

<!-- more -->

## 为什么 Go 做不到真正的热更新

解释型语言能做"真热重载"，靠的是运行时本身留了后门：Python 的 `importlib.reload` 能把一个已经导入的模块对象原地替换掉，Node 清掉 `require` 缓存重新 `require` 一遍也是同样的思路——程序运行的时候，代码本身是可以在运行时被替换的对象。

Go 完全是另一套模型：`go build` 产出的是一个静态链接的二进制文件，函数地址在编译期就已经定死，进程运行起来之后没有任何官方支持的方式能让你在不重启这个进程的前提下换掉某个函数的实现。想让改动生效，唯一的路径就是重新编译、重新启动整个进程——这是 Go 编译模型本身决定的，不是工具链偷懒没做，`air` 这类工具从设计第一天起就没打算装作能绕开这个限制。

## air：监听、编译、重启这三步怎么落地

```bash
go install github.com/air-verse/air@latest
#     air-verse/air 是目前维护活跃的官方仓库（原 cosmtrek/air 已迁移过去）
#     要求 Go 1.25 及以上

air init
#     在项目根目录生成一份可编辑的 .air.toml，不加这一步 air 也能跑，只是用内置默认配置
```

底层监听文件变化用的是 `github.com/fsnotify/fsnotify`（air 的 `go.mod` 里锁定的是 v1.9.0），跨平台的文件系统事件都由这个库抹平。基础配置只需要认识几个字段：

```toml
[build]
cmd = "go build -o ./tmp/main ."
# 编译出来的二进制放哪
bin = "tmp/main"
# 只监听这些扩展名的文件变化
include_ext = ["go", "tpl", "tmpl", "html"]
# 这些目录改动不触发重新编译
exclude_dir = ["assets", "tmp", "vendor", "frontend/node_modules"]
```

存一次 `.go` 文件，`air` 就会重新走一遍 `go build` 再重启进程，本地开发不用自己敲 `go run` 了。

## 不是所有改动都该触发完整重新编译

一个真实项目里，`.go` 文件之外经常还有前端资源、`templ` 模板、`sqlc` 生成的代码这类东西——改一行 CSS 也去触发一次完整的 `go build` 完全是浪费。`air` 用 `[[build.rules]]` 解决这个问题：匹配到规则的文件改动只跑规则自己的命令，不触发主体的重新编译和重启：

```toml
[build]
cmd = "go build -o ./tmp/main ."
exclude_dir = ["web"]

[[build.rules]]
name = "assets"
include_dir = ["web"]
include_ext = ["js", "ts", "css"]
cmd = "npm run build"

[[build.rules]]
name = "templ"
include_ext = ["templ"]
cmd = "templ generate"
```

如果某条规则生成的文件本身又是主体构建会监听的类型（比如 `templ generate` 生成 `.go` 文件），那条链路会自然衔接上——规则命令跑完之后，生成的 `.go` 文件变化会正常触发一次真正的重新编译，不需要手动再干预一次。

## `send_interrupt` 默认是 false：和优雅关闭的冲突

这是本文最值得记的一点。`air` 每次重新编译完，都要把旧进程干掉再启动新的——官方示例配置 `air_example.toml` 里这个字段的默认值是：

```toml
# Send Interrupt signal before killing process (ignored on Windows; uses TASKKILL)
send_interrupt = false
```

默认是 `false`，意味着 `air` 默认直接强制结束旧进程，不会先给它发一个"请退出"的信号。这里的冲突在于：如果你的服务按照标准做法接了 `signal.NotifyContext` 捕获 `SIGINT`/`SIGTERM`，配合 `http.Server.Shutdown()` 做[优雅关闭](/Go优雅关闭-HTTP服务与K8s滚动发布的隐藏竞态)，这套逻辑在 `air` 的默认配置下完全用不上——`air` 根本不会给这个信号，直接杀。本地开发时这通常无伤大雅（反正是本机调试），但如果你想让热重载期间也走一遍真实的优雅关闭路径（验证清理逻辑本身有没有 bug、观察正在处理的请求是不是真的等到了），就得显式打开这个选项：

```toml
[build]
send_interrupt = true
# 打开之后，air 会先发 Interrupt 信号，给进程一个体面退出的机会
kill_delay = 500
# 单位是纳秒，发送 Interrupt 之后等待这么久，还没退出才真正强杀
```

`kill_delay` 的单位需要留意一下——官方配置文件里写的是纳秒（`nanosecond`），不是更符合直觉的毫秒，默认值 `500` 纳秒短到几乎可以忽略不计，如果想让进程有实际可用的清理窗口，这个数值要按纳秒的量级去调（比如 5 秒对应 `5000000000`），直接抄一个"看起来合理"的小数字大概率不够用。

## `[proxy]`：重新编译完，浏览器自己刷新

写后端渲染页面（模板 + `net/http`，不是前后端分离的 SPA）的时候，`air` 还能顺带把浏览器手动刷新这一步也省掉：

```toml
[proxy]
enabled = true
proxy_port = 8090
# 浏览器打开这个端口，而不是应用自己监听的端口
app_port = 8080
```

原理不复杂：`air` 在应用前面搭一个小代理，浏览器改成访问 `proxy_port`，请求原样转发到 `app_port`；只要响应是 HTML，`air` 会在 `</body>` 标签之前注入一小段脚本，重新编译成功之后，这段脚本负责触发页面刷新。用这个功能有两个前提容易被忽略：页面必须有 `</body>` 标签，没有的话没地方注入脚本，页面会原样返回、刷新不会发生；改动的静态资源本身也得被 `include_dir`/`include_ext` 覆盖到，不然文件变了但 `air` 根本没监听到，自然也不会触发刷新。如果应用本身启动慢（要连数据库、加载一堆配置），代理会报"unable to reach app"，把 `app_start_timeout`（默认 5000 毫秒）调大就行。

## 用它，但别对它有过高期待

`air` 解决的是"改完代码不用手动敲命令重启"这一件事，效率提升是实打实的，但它从头到尾都是重新编译加重启这一套，不是真热更新——这个心理预期摆正了，`send_interrupt` 默认强杀旧进程这类行为就不会显得意外。本地嫌麻烦可以放着默认配置不管，但凡涉及验证优雅关闭逻辑本身对不对，记得先把 `send_interrupt` 打开，不然测的从来都不是你以为在测的那条路径。
