---
title: 'goland: 远程调试正在运行中的程序'
date: 2024-10-29 10:45:57
categories: golang
tags:
 - goland
 - remote debug
---

# 背景

某次逛`goland`的官网时，发现`goland`支持远程调试正在运行中的程序。然后闲暇之余，就想着试试。体验下来后，发现虽然有一些小的限制，但整体使用下来结合`goland`的gui界面，比纯打log调试还是舒服一些的。
![](../images/goland-remote-debug.jpg)
<!-- more -->

# 具体步骤
1. 首先注意项目使用的是不是`gopath`模式，如果是的话，客户端和远程主机需要在相同路径下进行打包编译，比如：
   1. 远程主机：`$GOPATH/src/debuggingTutorial/`
   2. 客户端：  `$GOPATH/src/debuggingTutorial/`
2. 编译： `go build -gcflags="all=-N -l" -o myApp` , myApp为编译后的文件名,可以随意替换。
     * 这里一定要是`-gcflags="all=-N -l"`，否则无法调试。
3. 在远程主机上运行Delve，有两种方式：
   1. 直接运行，这种是远程主机没有运行对应的服务时，启动： `dlv --listen=:2345 --headless=true --api-version=2 exec ./myApp`. 如果服务需要传入配置参数，可以这样加`-- --config=/path/to/config/file`
   2. attach到正在运行的服务上，这种是远程主机已经运行对应的服务，需要attach到正在运行的服务上，启动： `dlv --listen=:2345 --headless=true --api-version=2 attach <PID>`。 其中pid是正在运行的服务进程id。注意： 正在运行的服务必须要打开调试信息。
4. 在goland中配置remote调试：
   1. 打开goland，点击`Run` -> `Edit Configurations`
   2. 点击`+`，添加一个`Remote`
   3. 配置`Host`为远程主机的ip
   4. `Port`为2345
   5. `Debugger command`为`dlv`
   6. 点击`OK`
<span style="color:green;">[官方GIF详情参考](https://resources.jetbrains.com/help/img/idea/2024.2/go_create_the_remote_run_debug_configuration.animated.gif)</span>
1. 在goland中设置断点，然后点击`Debug`，就可以调试了。
<span style="color:green;">[官方GIF详情参考](https://resources.jetbrains.com/help/img/idea/2024.2/go_start_the_debugging_process_on_the_client_computer.animated.gif)</span>



[Goland官网文档参考](https://www.jetbrains.com/help/go/attach-to-running-go-processes-with-debugger.html#attach-to-a-process-on-a-remote-machine)


# 常见问题

## 编译的服务没有打开调试信息

这种情况常见于attach正在运行的服务时，可能之前服务编译打包的时候，没有打开调试信息，所以导致attach，无法调试。
我就是这种情况，以为加了调试信息。。。。（一般生产环境都是采用`-ldflags="-w -s"`关闭调试，压缩包大小的。）


## linux限制attach的权限

当你运行`dlv --listen=:2345 --headless=true --api-version=2 attach <PID>`时，可能会遇到这个错误：
```markdown
2024-10-28T17:00:57+08:00 warning layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)
Could not attach to pid 3279640: this could be caused by a kernel security setting, try writing "0" to /proc/sys/kernel/yama/ptrace_scope
```
这是因为linux限制了attach的权限。

解决办法：
1. 临时解决：
```bash
sudo echo 0 > /proc/sys/kernel/yama/ptrace_scope

# 或者
sudo sysctl -w kernel.yama.ptrace_scope=0
```
2. 永久解决，编辑 `/etc/sysctl.d/10-ptrace.conf` 文件（如果不存在则创建），添加以下内容：
```bash
kernel.yama.ptrace_scope = 0
```
然后执行 `sudo sysctl -p /etc/sysctl.d/10-ptrace.conf` 使配置生效。

> ptrace_scope 的值含义：
> 0: 所有进程都可以使用 ptrace
> 1: 只有父进程可以对子进程使用 ptrace（默认值）
> 2: 只有管理员可以使用 ptrace
> 3: 没有进程可以使用 ptrace
>
> 建议：如果是在开发环境中，可以设置为 0；如果是在生产环境，建议保持默认值 1 以保证安全性。
> 修改完成后，你就可以重新运行 dlv attach 命令了。需要注意的是，如果是临时解决方案，系统重启后需要重新设置。


## 其余常见情况

1. 检查 dlv 是否正确监听在 2345 端口： `netstat -anp | grep 2345`
2. 确认goland中断点设置是否正确：
   * 确保你的断点设置在会被实际执行到的代码行上
   * 检查断点是否真的被激活（在GoLand中显示为红色而不是灰色）
   * 尝试在 main() 函数开始处设置一个断点来测试
   * 检查是否触发了正确的代码路径，在关键位置加日志： `fmt.Println("debugging")`
3. 检查 GoLand 的调试配置：
   * 确保 Remote 配置中的 Host 和 Port 与 dlv 启动参数匹配
   * 检查 `Debugger port` 是否设置为 2345
   * 确认 `Host` 是否设置为 正确的对应ip
 4. 验证进程附加状态:
    1. 检查 dlv 进程状态： ` ps aux | grep dlv`
    2. 检查目标进程的调试状态: `cat /proc/<pid>/status | grep Trace`
 5. 检查防火墙设置： `sudo iptables -L | grep 2345`
 6. 检查远程主机是否安装了`delve`

