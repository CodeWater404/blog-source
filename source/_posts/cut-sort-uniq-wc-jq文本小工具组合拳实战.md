---
title: "cut、sort、uniq、wc、jq 实战：小工具组合怎么顶一个 awk 脚本"
date: 2026-07-14 10:11:31
cover: /img/p45.jpg
categories: tools
tags:
  - Linux
  - 命令行
  - CLI
  - 文本处理
description: "cut 截取列、sort 排序、uniq 去重计数、wc 统计、jq 处理 JSON，组合替代 awk 脚本。"
---

想知道一份访问日志里访问最多的 10 个 IP，不用现写一段 `awk` 脚本——`cut`、`sort`、`uniq`、`wc` 这几个小工具，管道接起来一行命令就能搞定，比现场想正则、调脚本快得多。

<!-- more -->

这是 Linux 命令系列的一篇，承接 [Linux 常见命令与工具地图](/Linux常见命令与工具地图)；这几个小工具经常和 [sed/awk](/sed与awk实战对比-文本处理工具怎么选) 配合甚至互相替代，简单场景用它们组合，比写一段 `awk` 脚本更直接。

## cut：按列或按字符截取

```bash
echo "alice,25,engineer" | cut -d',' -f1,3
#     -d：指定分隔符，这里是逗号
#     -f：选取第几个字段，1,3 表示第 1 和第 3 个，输出 alice,engineer

echo "hello world" | cut -c1-5
#     -c：按字符位置截取，1-5 表示第 1 到第 5 个字符，输出 hello
```

`cut` 的局限也很明显：分隔符必须是单个固定字符，遇到连续多个空格分隔的列（比如 `ps aux` 的输出），`cut -d' '` 会因为空格数量不固定而截错位置，这种场景交给 `awk` 的字段拆分更靠谱。

## sort：排序

```bash
sort file.txt
#     默认按字典序（ASCII）排序——数字字符串会按"字符"比较，"10" 排在 "2" 前面，容易踩坑

sort -n file.txt
#     -n：按数值大小排序，这时候 "2" 才会排在 "10" 前面

sort -r file.txt
#     -r：逆序排序，配合 -n 就是数值从大到小

sort -k2,2n file.txt
#     -k2,2：只按第 2 列排序（不写结束列会一直排到行尾，是很多人忽略的细节）
#     n 后缀表示这一列按数值比较

sort -u file.txt
#     -u：排序的同时去重，等价于 sort | uniq，但只需要一次调用
```

`sort` 默认是字典序这件事经常被忽略——处理纯数字列却忘了加 `-n`，排序结果看着"乱"，其实是没按数值比较。

## uniq：去重和统计重复次数

```bash
sort file.txt | uniq
#     去除相邻的重复行——注意 uniq 只对相邻的重复行生效，输入必须先排序过

sort file.txt | uniq -c
#     -c：统计每一行出现的次数，输出前面带一个计数
```

`uniq` 最容易踩的坑：它不会去重整个文件里所有重复的行，只会去除挨在一起的重复行。原始文件里两个相同的行如果隔着别的内容，`uniq` 根本发现不了——这就是为什么几乎所有 `uniq` 的用法前面都要先接一个 `sort`，把相同的内容排到一起，`uniq` 才能真正生效。

## wc：统计行数、单词数、字节数

```bash
wc -l file.txt
#     -l：统计行数，判断一个文件/输出有多少条记录时最常用

wc -w file.txt
#     -w：统计单词数（按空白字符分隔）

wc -c file.txt
#     -c：统计字节数
```

`grep -c` 和 `wc -l` 经常配合使用：`grep "ERROR" file.txt | wc -l` 数出日志里报错的行数，比自己写循环计数快得多。

## jq：命令行的 JSON 处理器

处理 API 响应或者 JSON 格式的日志，`jq` 是命令行下最常用的工具：

```bash
echo '{"name":"alice","age":25}' | jq '.name'
#     取单个字段，输出 "alice"（带引号，因为 jq 输出的是 JSON 值）

echo '[{"name":"alice"},{"name":"bob"}]' | jq '.[].name'
#     .[]：遍历数组里的每一项
#     .name：取每一项的 name 字段，依次输出 "alice" 和 "bob"

echo '[{"name":"alice","age":25},{"name":"bob","age":17}]' | jq '.[] | select(.age >= 18)'
#     select：按条件过滤数组，只保留满足条件的项，这里是只保留 age >= 18 的记录
```

`jq` 的语法本身是一门小型查询语言，能做的远不止取字段——排序、分组、重新构造 JSON 结构都支持，日常处理 API 响应，这几个基本用法已经能覆盖大部分场景。

## 组合实战：统计访问日志里最高频的 IP

访问日志每行开头是客户端 IP，统计出现次数最多的前 10 个：

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
#     awk '{print $1}'：取每行第一列，也就是 IP 地址
#     sort：排序，让相同的 IP 排到一起（uniq 生效的前提）
#     uniq -c：统计每个 IP 出现的次数
#     sort -rn：按次数数值从大到小排序
#     head -10：只看前 10 条
```

五个步骤，每一步只做一件很简单的事，串起来就完成了一个不算简单的统计任务——这正是 Unix 管道哲学的直接体现，也是[之前 xargs 那篇](/xargs管道与Linux命令设计哲学)提到的"小工具、文本接口、自由组合"的具体例子。

## 常用参数速查表

| 命令/参数 | 作用 |
|---|---|
| `cut -d -f` | 按分隔符截取指定字段 |
| `cut -c` | 按字符位置截取 |
| `sort -n` | 按数值大小排序，默认是字典序 |
| `sort -r` | 逆序排序 |
| `sort -k N,N` | 只按第 N 列排序 |
| `sort -u` | 排序同时去重 |
| `uniq -c` | 统计相邻重复行的次数，使用前必须先 sort |
| `wc -l` | 统计行数 |
| `wc -w` | 统计单词数 |
| `wc -c` | 统计字节数 |
| `jq '.field'` | 取 JSON 字段 |
| `jq '.[]'` | 遍历 JSON 数组 |
| `jq 'select(cond)'` | 按条件过滤 |

## 写到这里

这几个工具单独看都很简单，价值在组合——`sort` 配 `uniq` 做去重统计，`cut` 配 `wc` 做列提取加计数，`jq` 单独处理 JSON。遇到"想临时统计点什么"的需求，先想想能不能用这几个小工具接一条管道解决，很多时候比打开编辑器写脚本更快。
