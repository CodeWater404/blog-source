---
title: "grep：正则匹配、上下文控制与 -E/-P 实战"
date: 2026-07-11 10:00:00
cover: /img/p32.jpg
categories: tools
tags:
  - Linux
  - grep
  - 正则表达式
  - 命令行
description: "grep 的 BRE/-E/-P 三种正则模式区别在哪、-A/-B/-C 上下文控制怎么用、配合 xargs 批量处理匹配文件，一篇讲清楚原生 grep。"
---

本地终端习惯了用 ripgrep，SSH 到一台精简过的服务器或者容器镜像上，才发现根本没装 `rg`，只能用回系统自带的 grep。这时候才发现自己其实记不住 grep 的参数——`-E` 和 `-P` 到底有什么区别、括号要不要转义、上下文参数是哪三个字母。

grep 是所有 Linux 发行版都自带的工具，不需要额外安装，这篇把它自己的用法讲清楚。

<!-- more -->

这是 Linux 常见命令系列第二篇详细教程，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)。如果你的环境能装第三方工具，[ripgrep 通常是更好的选择](/终端搜索三件套-fzf-ripgrep-fd)——但装不上的场合，还得靠这篇。

## 三种正则模式：BRE、-E、-P 到底有什么区别

这是 grep 里最容易搞混的地方，因为同一个符号在不同模式下含义不一样。

**默认模式是 BRE（Basic Regular Expression，基础正则）**：`+`、`?`、`|`、`(`、`)`、`{`、`}` 这几个符号**不是**特殊字符，是普通文本本身；要让它们起特殊作用，得加反斜杠转义成 `\+`、`\?`、`\|`。这一点非常反直觉——很多人写 `grep "a+b"` 以为在匹配"一个或多个 a 后面跟 b"，实际在 BRE 下匹配的是字面的字符串 `a+b`。

**`-E` 启用 ERE（Extended Regular Expression，扩展正则，等价于老命令 `egrep`）**：上面那几个符号反过来变成特殊字符，不用转义就能用：

```bash
grep 'a\+b' file.txt
#     BRE 默认模式：\+ 转义后才是"一个或多个 a"，匹配 ab、aab、aaab...

grep -E 'a+b' file.txt
#     -E 扩展正则：+ 天生就是特殊字符，效果和上面那条一样
#     -E 等价于直接用 egrep 命令（egrep 只是 grep -E 的别名，新脚本建议用 -E）
```

**`-P` 启用 PCRE（Perl 兼容正则）**：语法最丰富，`\d`（数字）、`\s`（空白）、`\w`（单词字符）、`\b`（单词边界）、非贪婪匹配 `.*?` 这些 BRE 和 -E 都不支持的写法，只有 -P 能用：

```bash
grep -P '\d{3}-\d{4}' contacts.txt
#     -P：Perl 正则，\d{3} 表示连续 3 个数字，匹配形如 123-4567 的号码
#     BRE/-E 下要写成等价的 [0-9]\{3\} 或 [0-9]{3}，没有 \d 这种简写
```

同一个"匹配一个或多个数字"的需求，在三种模式下写法完全不同：

| 需求 | BRE（默认） | `-E` | `-P` |
|---|---|---|---|
| 一个或多个数字 | `[0-9]\+` | `[0-9]+` | `\d+` |
| 括号分组 | `\(abc\)` | `(abc)` | `(abc)` |
| 或（多选一） | `a\|b`（GNU 扩展，非 POSIX 标准） | `a\|b` | `a\|b` |

**一个容易踩的坑**：`-P` 依赖 grep 编译时链接了 PCRE 库，绝大多数主流发行版的 grep 都带这个支持，但个别精简过的容器镜像可能没有，报 `grep: this platform does not support -P` 时就只能退回 -E 或 BRE 手动转义。日常写脚本，如果不确定目标机器的 grep 支持情况，优先用 `-E`——兼容性最好，语法也不用记转义规则。

## 基础用法：递归、忽略大小写、反向匹配

```bash
grep -r "TODO" src/
#     -r：递归搜索目录下所有文件

grep -R "TODO" src/
#     -R：和 -r 的唯一区别是符号链接的处理，见下面的说明

grep -i "error" app.log
#     -i：忽略大小写，ERROR、Error、error 都能匹配到
```

**先说清楚一个前提**：不加 `-r`/`-R`，grep 根本不会进目录递归——不管这个参数是真实目录还是指向目录的符号链接，直接报错 `grep: xxx: Is a directory`，跳过不搜。要进目录搜索，`-r` 是必须的，"跟不跟随符号链接"是加了 `-r` 之后才谈得上的下一层问题：

- **`-r`**：命令行里直接传的符号链接（比如 `grep -r "TODO" src/shared-link`，`shared-link` 指向另一个目录）会被跟随，当成真目录进去搜；但递归过程中、在 `src/` 内部某个子目录里才碰到的符号链接，不会跟随，直接跳过
- **`-R`**：不区分是不是命令行直接给的，只要递归时遇到符号链接指向目录就都跟随进去搜

如果符号链接指向的是**文件**而不是目录，这个规则完全不适用——不需要 `-r`，直接 `grep "pattern" 符号链接文件` 就能读到内容，因为这只是普通的文件读取，操作系统打开文件时本来就会跟随符号链接，跟目录递归是两回事。

```bash
grep -v "DEBUG" app.log
#     -v：反向匹配，输出不包含 DEBUG 的行——排查日志时过滤掉噪音行常用
```

## 只要结果的一部分：-l、-L、-c、-o、-n

默认 grep 输出的是匹配到的整行，但很多时候你只想要文件名、数量，或者匹配的那一小段：

```bash
grep -l "TODO" src/**/*.go
#     -l：只列出包含匹配内容的文件名，不显示具体是哪一行

grep -L "TODO" src/**/*.go
#     -L：反过来，列出不包含匹配内容的文件名

grep -c "ERROR" app.log
#     -c：只输出匹配的行数，不显示内容本身，快速确认量级用

grep -o -E '[0-9]+\.[0-9]+\.[0-9]+' package.json
#     -o：只输出匹配到的那部分文本，不是整行；这里配合 -E 提取版本号

grep -n "func main" main.go
#     -n：匹配的每一行前面带上行号，定位到编辑器里直接跳转用
```

## 精确匹配：-w 整词、-x 整行

不加限制的话，`grep "cat"` 会把 `category`、`concatenate` 这类包含 "cat" 子串的词也匹配上：

```bash
grep -w "cat" animals.txt
#     -w：整词匹配，只匹配独立的单词 cat，不会命中 category 里的 cat

grep -x "done" status.txt
#     -x：整行匹配。status.txt 里某一行如果内容恰好就是 "done" 这四个字符，能匹配上；
#     如果那一行是 "task done" 或者 "done!"，虽然包含 "done" 这个子串，但整行内容不完全等于 "done"，匹配不上
```

## 上下文控制：-A、-B、-C

排查报错时，只看到匹配的那一行往往不够，前后几行的上下文才能定位问题：

```bash
grep -A 3 "Exception" app.log
#     -A（after）：匹配行之后再多输出 3 行，看报错发生后系统还做了什么

grep -B 3 "Exception" app.log
#     -B（before）：匹配行之前再多输出 3 行，看报错发生前的调用链

grep -C 3 "Exception" app.log
#     -C（context）：等价于同时加 -A 3 -B 3，前后各多输出 3 行
```

多个不连续的匹配段之间，grep 会用 `--` 分隔线隔开，方便肉眼区分不同的匹配区块。

## 按文件类型过滤：--include、--exclude、--exclude-dir

递归搜索整个项目目录时，通常不想搜到 `node_modules`、构建产物这些无关内容：

```bash
grep -r "TODO" . --include="*.go"
#     --include：只在匹配这个 glob 模式的文件里搜，这里只搜 .go 文件

grep -r "TODO" . --exclude="*.min.js"
#     --exclude：排除匹配这个模式的文件，压缩后的 JS 文件通常没必要搜

grep -r "TODO" . --exclude-dir="node_modules" --exclude-dir=".git"
#     --exclude-dir：整个目录都跳过，node_modules 和 .git 是最常见的两个
```

这几个选项只在配合 `-r`/`-R` 递归搜索时才有意义，搜单个文件用不上。

## 和管道、xargs 组合的实战

grep 真正的威力在于能接进管道，和其他命令组合：

```bash
grep -l "deprecated_func" src/**/*.go | xargs grep -n "deprecated_func"
#     先用 -l 找出所有包含 deprecated_func 的文件（只要文件名）
#     再用 xargs 把这些文件名传给第二个 grep，这次带上 -n 看具体在哪一行
#     两步拆开的好处：第一步先确认"影响范围有多大"，再决定要不要继续深入

grep -r "TODO" . --include="*.go" | grep -v "TODO(done)"
#     链式过滤：第一个 grep 找出所有 TODO，第二个 grep 用 -v 排除已经标记完成的
#     两个 grep 串联，比写一条复杂的正则更好读

grep -l "old_config_key" config/*.yaml | xargs sed -i 's/old_config_key/new_config_key/g'
#     场景：配置项改名了（old_config_key 要统一换成 new_config_key），但不确定散落在多少个 yaml 文件里
#     grep -l 先扫一遍，只列出真正包含 old_config_key 的文件名（没提到的文件完全不动）
#     xargs 把这些文件名接力传给 sed，对每个文件执行替换，一条命令完成"找出范围 + 批量改名"两件事
#     -i：sed 的原地修改参数，不加的话 sed 默认只输出到 stdout，不会真的改文件内容
```

`grep -l | xargs` 这个组合的价值在于分两步走：第一步只是"侦察"，看看改动会影响哪些文件，确认范围没问题了再执行第二步的实际操作，比直接一条命令梭哈更安全。

## grep 常用参数速查表

| 参数 | 作用 |
|---|---|
| `-r`, `--recursive` | 递归搜索目录，只跟随命令行直接给出的符号链接 |
| `-R`, `--dereference-recursive` | 递归搜索目录，跟随所有遇到的符号链接 |
| `-i`, `--ignore-case` | 忽略大小写 |
| `-v`, `--invert-match` | 反向匹配，输出不匹配的行 |
| `-l`, `--files-with-matches` | 只列出有匹配的文件名 |
| `-L`, `--files-without-match` | 只列出没有匹配的文件名 |
| `-c`, `--count` | 只输出匹配的行数 |
| `-o`, `--only-matching` | 只输出匹配到的部分，不是整行 |
| `-n`, `--line-number` | 显示匹配行的行号 |
| `-w`, `--word-regexp` | 整词匹配 |
| `-x`, `--line-regexp` | 整行匹配 |
| `-A NUM`, `--after-context=NUM` | 额外输出匹配行之后 NUM 行 |
| `-B NUM`, `--before-context=NUM` | 额外输出匹配行之前 NUM 行 |
| `-C NUM`, `--context=NUM` | 等价于 `-A NUM -B NUM` |
| `-E`, `--extended-regexp` | 启用扩展正则（等价于 `egrep`） |
| `-P`, `--perl-regexp` | 启用 Perl 兼容正则 |
| `--include=GLOB` | 只在匹配 GLOB 的文件里搜（配合 `-r`/`-R`） |
| `--exclude=GLOB` | 排除匹配 GLOB 的文件（配合 `-r`/`-R`） |
| `--exclude-dir=GLOB` | 排除匹配 GLOB 的目录（配合 `-r`/`-R`） |

## grep 之外

以上这些参数，ripgrep 全部支持而且默认行为更省心（自动递归、自动跳过 `.gitignore`），装得上就优先用 [ripgrep](/终端搜索三件套-fzf-ripgrep-fd)。Linux 命令系列下一篇计划写 `find`——参数量级比 grep 还大，文件查找的条件组合、`-exec` 的两种写法够写满一整篇。
