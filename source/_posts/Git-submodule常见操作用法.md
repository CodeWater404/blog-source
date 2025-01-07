---
title: 'Git: submodule常见操作用法'
date: 2025-01-07 14:45:29
categories: git
tags:
  - submodule
---

# 背景

某个工作中的项目需要包含并使用另一个项目，也许是第三方库，或者你独立开发的。 现在问题来了：你想要把它们当做两个独立的项目，同时又想在一个项目中使用另一个库。
`Git submodule` 就是为了解决这个问题而生的,它允许你将一个仓库作为另一个仓库的子模块,同时让这两个库保持独立的提交记录。
这里记录一下submodule常见操作用法，方便以后查阅。

<!-- more -->

# git submodule


## 1. 添加子仓库

要将一个现有的 Git 仓库添加为子仓库，可以使用 `git submodule add` 命令。

操作步骤：

```bash
git submodule add <repository_url> <path>
```

- `<repository_url>`：子仓库的 Git 仓库地址。
- `<path>`：子仓库在主仓库中的存放路径。

**示例**：

```bash
git submodule add https://github.com/example/libfoo.git libs/libfoo
```

## 2. 初始化子仓库

如果你克隆了一个包含子仓库的主仓库，你需要初始化子仓库。

操作步骤：

```bash
git submodule init
```

## 3. 更新子仓库

初始化后，你需要更新子仓库以获取其内容。

操作步骤：

```bash
git submodule update
```

> 当然，上面两个步骤可以合并为一个命令： `git submodule update --init`


## 4. 查看子仓库状态

要查看子仓库的状态，可以使用以下命令：

```bash
git submodule status
```

## 5. 更新子仓库到最新版本

如果子仓库有更新，你可以进入**子仓库目录并拉取最新的更改**。

操作步骤：

```bash
cd <path_to_submodule>
git pull origin master
```

## 6. 提交对子仓库的更改

如果你在子仓库中做了更改并希望将其提交到主仓库，你需要先在子仓库中提交更改，然后在主仓库中更新子仓库的引用。

操作步骤：

1. 在**子仓库**中提交更改:

```bash
git add .
git commit -m "Update submodule"
```

2. 返回**主仓库**并更新子仓库引用:

```bash
cd ..
git add <path_to_submodule>
git commit -m "Update submodule reference"
```

## 7. 删除子仓库

如果你想要删除子仓库，可以按照以下步骤操作：

1. 从主仓库中删除子仓库的引用：

```bash
git rm --cached <path_to_submodule>
```

2. 删除子仓库的目录：

```bash
rm -rf <path_to_submodule>
```

3. 提交更改：

```bash
git commit -m "Remove submodule"
```

## 8. 克隆包含子仓库的主仓库

如果你想要克隆一个包含子仓库的主仓库，可以使用以下命令：

```bash
git clone --recurse-submodules <repository_url>
```

如果已经克隆了主仓库，可以使用以下命令初始化和更新子仓库：

```bash
git submodule update --init --recursive
```

# `git submodule update` 和在子仓库中执行`git pull origin master` 的区别？


如读者所见，上面关于子仓库的更新有两种方式：
- update：使用 `git submodule update` 命令更新子仓库
- pull：在子仓库中执行 `git pull origin master` 命令更新子仓库

那这两种区别是啥？以下是这两个命令的详细比较：

## `git submodule update`

### 目的
- `git submodule update` 用于将子模块的状态<u>更新到主仓库所记录的特定提交</u>。这意味着它会**将子模块检出到主仓库中记录的提交哈希值**。

### 行为
- 当你在主仓库中执行 `git submodule update` 时，Git 会检查 `.gitmodules` 文件和主仓库的索引，找到每个子模块的最新提交哈希值，并将子模块的工作目录更新到这些特定的提交。
- 如果子模块的工作目录中有未提交的更改，`git submodule update` 不会影响这些更改，但它会将子模块的 HEAD 指针移动到主仓库所记录的提交。
- 另外，update这个命令并不需要在子模块中执行，它也可以在主仓库中执行。



## `git pull origin master` 在子仓库中执行

### 目的
- `git pull origin master` 用于从远程仓库拉取最新的提交并合并到当前分支。这是一个常规的 Git 操作，用于更新子模块的内容。
- pull这个命令必须在子模块中执行，才是对子仓库的更新；如果在主仓库中执行，那么就是对主仓库的更新。

### 行为
- 当你在子模块中执行 `git pull origin master` 时，Git 会从远程仓库拉取最新的提交，并将这些提交合并到当前分支。这可能会导致子模块的 HEAD 指针移动到远程仓库的最新提交。
- 这个操作会改变子模块的状态，并可能导致主仓库中的子模块引用与子模块的实际状态不一致。

实际上就是说：主仓库下的子仓库的状态并不一定与子仓库的最新状态保持一致！


## 总结

- **`git submodule update`**：将子模块更新到主仓库所记录的特定提交，保持主仓库和子模块之间的引用一致。
- **`git pull origin master`**：从远程仓库拉取最新的提交并合并到当前分支，可能导致子模块的状态与主仓库中的引用不一致。

## 使用场景

- 如果你想确保子模块的状态与主仓库的引用一致，使用 `git submodule update`。
- 如果你想更新子模块到远程仓库的最新状态，并且不介意主仓库中的引用可能变得不一致，可以使用 `git pull origin master`。


