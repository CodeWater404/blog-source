---
title: "tar、zip/unzip：归档压缩命令与常见坑实战"
date: 2026-07-11 17:00:00
cover: /img/p34.jpg
categories: tools
tags:
  - Linux
  - tar
  - zip
  - 命令行
description: "tar 为什么不等于压缩、-z/-j/-J 怎么选、zip/unzip 的跨平台场景，配合真实踩坑讲清楚 Linux 归档压缩命令的实战用法。"
---

部署脚本里要把一堆文件打包传到服务器，敲命令时永远在纠结：`tar -czf` 和 `tar -xzf` 哪个是打包哪个是解包？参数顺序换一下会不会报错？打包完发现体积比预期大一倍，是哪里出了问题？

这些问题的根源都是同一件事：`tar` 到底在干什么，很多人从来没搞清楚过。

<!-- more -->

这是 Linux 常见命令系列第四篇详细教程，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)、[chmod/chown](/Linux权限与属主管理-chmod-chown实战)、[grep](/grep正则匹配与上下文控制实战)、[find](/find条件组合与exec性能实战)。

## tar 是什么：打包和压缩是两件事

`tar` 全称 tape archive（磁带归档），这个名字暴露了它的历史——最早是为了把多个文件写进磁带这种只能顺序读写的设备设计的。它做的事情只有一件：**把多个文件和目录归档成一个文件流**，文件之间的边界、权限、目录结构都保留下来，但内容本身**不压缩**。

压缩是另外一回事，靠 `tar` 调用外部的压缩程序（gzip、bzip2、xz）来完成。这就是为什么打包压缩的命令永远长这样：

```bash
tar -czf archive.tar.gz files/
#     -c：创建归档（打包）
#     -z：调用 gzip 压缩
#     -f：指定输出文件名
#     三个动作一步做完，但概念上是两件事：先打包，再压缩
```

只打包不压缩，得到的是 `.tar` 文件；打包之后再压缩，才是 `.tar.gz`、`.tar.bz2` 这些格式——文件名里的双重后缀不是巧合，就是在诚实地告诉你这个文件经过了两道工序。

## 核心操作模式：-c、-x、-t、-r

四个字母决定 tar 这次要做什么：

```bash
tar -czf archive.tar.gz files/
#     -c（create）：创建一个新归档

tar -xzf archive.tar.gz
#     -x（extract）：把归档解出来

tar -tzf archive.tar.gz
#     -t（list）：只列出归档里有什么文件，不真的解压
#     解压一个来路不明的压缩包之前，先用这个看看里面到底装了什么，避免解出一堆文件把当前目录搞乱

tar -rf archive.tar files/new_file.txt
#     -r（append）：往已有的归档里追加文件
#     注意 -r 不能配合压缩格式使用（不能对已压缩的归档直接追加），只在处理未压缩的 .tar 文件时有效，不算高频用法
```

`-c`、`-x`、`-t`、`-r` 这四个是互斥的，一条命令只会用其中一个——决定这次操作"创建""解压"还是"查看"。

## 压缩格式选择：-z、-j、-J

打包之后要不要压缩、用哪种压缩算法，是三个独立的选项：

```bash
tar -czf archive.tar.gz files/
#     -z：gzip 压缩，速度快，压缩率一般，日常打包首选

tar -cjf archive.tar.bz2 files/
#     -j：bzip2 压缩，压缩率比 gzip 更高，但压缩速度更慢

tar -cJf archive.tar.xz files/
#     -J：xz 压缩，压缩率最高，速度也最慢，适合追求体积、不在乎时间的场景（比如长期归档存储）
```

选型建议很简单：不确定就用 `-z`，这是使用最广泛、兼容性最好的选择；只有明确要求"体积越小越好、时间不是问题"时才考虑 `-J`；`-j` 处在中间地带，除非有历史包袱（老脚本已经在用 bzip2），日常新写的命令不太有必要专门选它。

这三个选项本质上是 tar 去调用系统里对应的外部压缩程序（gzip、bzip2、xz），不是 tar 自己内置的压缩实现——精简过的容器镜像如果没装对应的二进制，加了 `-z` 也会报错，排查"tar 报错找不到命令"时先确认这个压缩程序本身装了没有。

## `-f` 参数：几乎总是需要

`-f` 用来指定归档文件名，几乎每条 tar 命令都会带上——不写 `-f`，tar 默认会尝试读写标准输入输出而不是一个文件，这在日常场景下基本不是你想要的行为：

```bash
tar -czf backup.tar.gz /var/www/site
#     -f 后面紧跟的 backup.tar.gz 就是输出文件名

tar -zcf backup.tar.gz /var/www/site
#     -czf 和 -zcf 效果一样——c、z 谁前谁后无所谓，因为它俩都不需要额外参数

tar -fcz backup.tar.gz /var/www/site
#     错误示范：-f 一旦不在最后一位，紧跟在它后面的 cz 会被当成 -f 的参数（文件名变成字面的 "cz"），
#     真正的文件名 backup.tar.gz 反而落空，GNU 官方文档明确警告：重排选项顺序时不把对应参数一起挪走，
#     可能导致覆盖文件——把 -f 放最后是唯一安全的写法
```

## 解压到指定目录：-C

不需要先 `cd` 到目标目录再解压，`-C` 直接指定解压目的地：

```bash
tar -xzf archive.tar.gz -C /opt/deploy/
#     -C：解压到这个目录，不影响当前工作目录
#     等价于 cd /opt/deploy/ && tar -xzf /path/to/archive.tar.gz，但不用你手动切目录、也不用担心相对路径算错
```

## 排除文件：--exclude

打包项目目录时，通常不想把 `.git`、`node_modules` 这类无关内容也打进去：

```bash
tar -czf project.tar.gz --exclude=".git" --exclude="node_modules" project/
#     --exclude：排除匹配的文件或目录，可以写多个
#     打包前排除比打包后再删省事得多，尤其是 node_modules 这种体积大的目录
```

## 只解压部分内容

归档很大、只需要其中一两个文件时，不用全解出来：

```bash
tar -xzf archive.tar.gz path/to/specific-file.txt
#     在 -f 后面直接跟归档内的具体路径，只解出这一个文件
#     具体路径要和 tar -tzf 列出来的路径完全一致，包括开头有没有 ./ 这种细节
```

## 两个常见坑

**打包时用了绝对路径，解压到别的机器上跑到意料之外的地方**：

```bash
tar -czf backup.tar.gz /home/user/project
#     打包进去的路径信息是 home/user/project（tar 默认会去掉开头的 /）
#     解压时会在当前目录下创建 home/user/project 这一整串目录结构，而不是直接解出 project 内容

cd /home/user/project && tar -czf backup.tar.gz .
#     正确做法：先 cd 进目标目录，再用相对路径（. 表示当前目录）打包
#     解压出来就是干净的文件内容，不会带着一层多余的目录结构
```

**压缩包体积比预期大，排查是不是重复压缩了**：图片、视频、已经是 `.zip`/`.gz` 格式的文件本身已经压缩过，信息熵接近上限，再用 `-z` 压一遍收益很小，有时因为压缩算法的元数据开销，体积甚至会比不压缩略大。如果打包对象里混了大量这类文件，可以考虑不加压缩参数、只用纯 `-c` 打包（`.tar` 而不是 `.tar.gz`），省下压缩耗时也不会有实质损失。

## zip、unzip：跨平台场景的另一个选择

tar 打包出来的格式在 Linux/macOS 生态里很自然，但要把文件发给用 Windows 的同事，对方系统原生就认识 `.zip`，不用额外装解压软件——这是 `zip`/`unzip` 的核心价值：跨平台兼容性比 tar 更好。

```bash
zip -r archive.zip project/
#     -r：递归打包整个目录，跟 tar 的行为类似
#     zip 和 tar 不同：打包和压缩是同一步完成的，没有"先打包再压缩"这道概念区分

zip -r archive.zip project/ -x "*.log"
#     -x：排除匹配的文件，这里排除所有 .log 文件

unzip archive.zip -d /target/dir
#     -d：解压到指定目录，效果类似 tar 的 -C

unzip -l archive.zip
#     -l：只列出压缩包里有什么文件，不真的解压，类似 tar -t
```

zip 还有一个 tar 没有的功能：直接加密。`zip -e archive.zip file1 file2` 会在打包时提示输入密码，之后解压需要密码才能打开。但这里有个安全上的坑要提醒：**标准 zip 加密的强度很弱**（用的是老旧的算法，容易被破解），只适合防"顺手翻一下"这种低要求场景，真正需要保密用 `gpg` 或 `age` 这类专门的加密工具，别指望 zip 密码扛得住认真的破解尝试。

## tar 常用参数速查表

| 参数 | 作用 |
|---|---|
| `-c` | 创建归档 |
| `-x` | 解压归档 |
| `-t` | 只列出归档内容，不解压 |
| `-r` | 追加文件到已有归档（不能用于已压缩的归档） |
| `-z` | 调用 gzip 压缩/解压，速度快 |
| `-j` | 调用 bzip2 压缩/解压，压缩率更高但更慢 |
| `-J` | 调用 xz 压缩/解压，压缩率最高，最慢 |
| `-f FILE` | 指定归档文件名 |
| `-v` | 显示处理过程中的文件列表（verbose） |
| `-C DIR` | 解压到指定目录 |
| `--exclude=PATTERN` | 排除匹配的文件或目录 |

`zip`/`unzip` 常用的几个参数：

| 参数 | 作用 |
|---|---|
| `-r` | 递归打包整个目录 |
| `-x PATTERN` | 排除匹配的文件 |
| `-e` | 加密压缩包（强度弱，别指望防真正的攻击者） |
| `-d DIR` | （unzip）解压到指定目录 |
| `-l` | （unzip）只列出内容，不解压 |

## tar 之外

系列下一篇计划写 `rsync`——增量同步、排除规则、远程传输这些参数量级比 tar 还大，够写满一整篇。
