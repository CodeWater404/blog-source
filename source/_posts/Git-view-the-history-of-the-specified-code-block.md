---
title: 'Git: 查看指定代码块的历史记录'
date: 2025-01-02 11:08:50
categories: git
tags:
  - log
  - blame
  - history
---

# 背景

某次找bug的时候，发现代码被改过了，所以想要看这块代码修改的历史记录。目前IDE右键是支持**文件的历史记录**的，但是不支持代码块级别查看。
当然有一些可能会有，例如`cursor`，但是使用起来就比较麻烦,而且也不是我先想要的效果。
***痛点在于***：cursor的行级别查看是可以，但是无法显示出所选行的所有历史记录，必须要选中某个commit，但实际上你如果出现我上面想要查看代码的情况，你是不知道错误的修改在哪次的commit里面，这样的话就导致你要多次重复的查看，很麻烦。

<!-- more -->


![](../images/git%20line%20history%20for%20cursor.png)
![](../images/git%20linie%20history%20examples%20for%20cursor.png)

# 解决

## git blame、 git log

既然IDE不支持，那么按照常规思路：
- 官方命令支持
- 通过脚本解决

我们先来看官方命令是否支持，经过搜索发现Git查看历史记录的有两种命令，分别是`git blame`和`git log`。

1. `git blame`: 逐行显示指定文件的每一行代码是由谁在什么时候引入或修改的。查看代码块的相关命令：
   1. `git blame -L <起始行号>,<结束行号> <文件>`：查看指定行号范围内的代码注释。
   2. 可以看到这根本不是想要的，*pass*。
2. `git log`: 用于查看 Git 仓库中提交历史记录。查看代码块的相关命令：
   1. `git -L <start>,<end>:<file>`: 查看某个文件指定行号范围的历史记录。
   2. `git -L:<funcname>:<file>`: 查看某个文件指定函数名的历史记录。
   3. 这两个命令符合想要的效果。

`git log -L:discover:chat.py`效果如下：
![](../images/git%20log%20-L.png)

行号的就不演示了，效果一样，可以看到历史commit这部分代码块的记录。
既然官方命令支持我们想要的效果，那么就不去折腾脚本这种方法了。

## `git log -L:<funcname>:<file>`缺点

在使用过程中，发现这个用法如果在某个文件中函数名第一次出现不是在声明函数的地方，那么显示的结果其实只是第一次出现相同字符串的地方，也就是说不能很好的识别真正函数的地方！

例如：
`export`第一次出现在`from src.export import ChatExporter, MarkdownChatFormatter, MarkdownFileSaver`,但我们实际上想要看的是`export`函数：

```python
@app.command()
def export(
    db_path: str = typer.Argument(None, help="The path to the SQLite database file. If not provided, the latest workspace will be used."),
    output_dir: str = typer.Option(None, help="The directory where the output markdown files will be saved. If not provided, prints to command line."),
    latest_tab: bool = typer.Option(False, "--latest-tab", help="Export only the latest tab. If not set, all tabs will be exported."),
    tab_ids: str = typer.Option(None, help="Comma-separated list of tab IDs to export. For example, '1,2,3'. If not set, all tabs will be exported.")
):
```
[代码例子](https://github.com/somogyijanos/cursor-chat-export)


> 参考：
> [官方文档](https://git-scm.com/docs/git-log)
>
> 如果你有更好的方法，换衣在评论区提出！
>


