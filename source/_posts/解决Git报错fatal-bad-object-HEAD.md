---
title: '解决Git报错fatal: bad object HEAD'
date: 2024-10-24 20:13:02
categories: problem
tags:
 - git
---

# 背景

某次工作时，在本地"dev"分支开发完，准备往本地"master"分支上合并。当我自信的敲下：
```shell
git checkout master
```
然后，问题出现了。。。。
```
# 这里隐藏相关文件名

error: Your local changes to the following files would be overwritten by checkout:
        **/*.go
        **/*.sh
        **/
        ...
Please commit your changes or stash them before you switch branches.
Aborting

# 中间有个交互，问我是否继续，我当时由于看到这一堆报错，就没看前面的信息，按下了“n”

# 然后再敲git命令时，就报了这个错,且大部分操作仓库的命令都无法使用。。。。
 git status
        fatal: bad object HEAD  

 git log
        fatal: bad object HEAD  
```

<!-- more -->



# 解决

## 方法一： 删除.git文件夹

`fatal: bad object HEAD`这个报错从表面意思来看就是一个错误的head对象，经过查资料之后，原因也大差不差，就是从head引用的分支指向了错误的提交对象。
至于为什么会在切分支的时候会引发这个错误，由于暂时没办法复现，无法深究。

在知道报错信息之后，查找解决方法，但是大部分的方案都是需要：

1. <font color="red">删除`.git`文件夹,重新init一个仓库。</font>
2. 或者直接回滚到远程分支所在的commit。

> [参考1](https://stackoverflow.com/questions/20264032/git-status-shows-fatal-bad-object-head) \
> [参考2](https://panjeh.medium.com/git-status-fatal-bad-object-head-af22724f48b9) \
> [参考3](https://www.swyx.io/solve-git-bad-object-head) \
> 其余不在列出。。。。


## 方法二： 回滚+恢复（推荐）

> 如果本地有未提交的commit，那么方法一是不适合的。因为相当于几天的工作白干了，而且大量的记录都丢失了。

> 在既需要保存本地未push的commit，又需要修复git仓库这种情况下，找了很多资料，没有找到想要的解决方案。
> 
> 最终，和涛哥（一位不愿透露姓名的大佬）的讨论之下，给了一个提示：想办法回滚到事故前的commit，然后利用reflog+cherry-pick恢复。

探索之下，详细操作如下：
1. 假设本地仓库`example` ,先备份一下本地仓库`tmp`，防止误操作导致的后果
2. 在a仓库操作：
   1. 回滚到和远程分支一样的状态： `git reset --hard origin/dev`
   2. 到这里，example库已经修复好了，测试git命令： `git log` ；但是本地未提交的commit还需要恢复！
   3. 利用`reflog`查看本地未push的是哪几个commit： `git reflog`。假设有这个几个commit： `... => a => b => c => d =>...` ，bc是本地未提交的commitId，a是回滚后所在的位置,d是别的操作产生的commitid。
   4. 恢复： `git cherry-pick a..c`。 因为`cherry-pick`不采用第一个开始的commit，所以从a开始，d不是我们想要恢复的commit，所以不包含。
   5. 如果有报错很正常，是冲突，解决一下对应的文件即可。
      1. 解决完冲突之后：`git add 相关文件` ,
      2. 接着 ` git cherry-pick --continue `即可。
3. 上述步骤操作完之后，如果对于之前恢复的commit不放心是不是有错误的代码，或者根本没回复，可以采用这个命令对比： ` git diff --no-index -- .\example\ .\tmp\ ` 。tmp就是之前还未修复的原始库备份。


![操作图解](../images/git%20bad-object-head.png)


