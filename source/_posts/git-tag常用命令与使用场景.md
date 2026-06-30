---
title: "git tag 常用命令与使用场景"
date: 2026-06-30 10:00:00
categories: tools
tags:
  - git
  - CLI
  - 版本管理
description: "git tag 完整教程：附注标签 vs 轻量标签的区别、打标签推送远端、detached HEAD 的坑、版本发布完整流程，一篇备查够用。"
---

tag 是给某个 commit 贴一个永久名字。最常见的用途是版本发布，打完一个版本就在这个 commit 上打个 `v1.2.0`，之后随时能找回来，也能在 GitHub 上直接生成 Release。

<!-- more -->

---

## 两种 tag：轻量标签 vs 附注标签

Git 有两种 tag，行为不同：

**轻量标签（lightweight）**：只是一个 commit 的别名指针，本质上和分支名差不多，但不会移动。不存储额外信息。

```bash
git tag v1.0.0
```

**附注标签（annotated）**：独立的 Git 对象，包含打标签者的名字、邮件、时间、tag message，还能用 GPG 签名。`git show` 能看到这些元信息。

```bash
git tag -a v1.0.0 -m "Release v1.0.0"
#      -a：annotated，创建附注标签（区别于默认的轻量标签）
#      -m：message，标签的说明文字，不加 -m 会弹出编辑器让你填写
```

**什么时候用哪个**：

- 正式版本发布 → 用附注标签（`-a`），信息完整，`git describe` 默认只识别附注标签
- 临时标记某个 commit（自用）→ 轻量标签够用

---

## 打标签

### 给当前 commit 打标签

```bash
# 轻量标签
git tag v1.0.0

# 附注标签（推荐用于版本发布）
git tag -a v1.0.0 -m "Release v1.0.0"
#      -a：annotated，创建附注标签
#      -m：message，填标签说明，省略此参数会弹出 $EDITOR 让你手动输入
```

### 给历史 commit 补打标签

先找到目标 commit 的 hash：

```bash
git log --oneline
#       --oneline：每个 commit 只显示一行（短 hash + 提交信息），方便找 hash
# a3f2c1d feat: add payment module
# 8b9e012 fix: resolve login timeout
```

然后指定 hash 打标签：

```bash
git tag -a v0.9.0 8b9e012 -m "Release v0.9.0"
#      -a：附注标签
#         v0.9.0：标签名
#                8b9e012：目标 commit 的 hash（不指定则默认当前 commit）
#                         -m：标签说明
```

---

## 查看标签

```bash
# 列出所有标签（按字母序）
git tag

# 通配符过滤，只看 v1.x.x 系列
git tag -l "v1.*"
#      -l：list，配合通配符（glob）过滤标签名，* 匹配任意字符

# 查看标签详情（附注标签会显示 tagger、日期、message；轻量标签只显示 commit 信息）
git show v1.0.0

# 查看标签指向哪个 commit（输出完整 hash）
git rev-parse v1.0.0
```

---

## 推送到远端

本地打的标签不会自动同步到远端，需要显式推送。

```bash
# 推送单个标签（和推送分支语法一样）
git push origin v1.0.0

# 推送所有本地标签，包括轻量标签
git push origin --tags
#               --tags：把本地所有 tag 一次性推送到远端

# 只推送附注标签，不推轻量标签（更安全）
git push origin --follow-tags
#               --follow-tags：只推送有对应 commit 的附注标签，轻量标签不推
```

日常版本发布推荐用 `--follow-tags`，只把正式的附注标签推上去。

---

## 删除标签

### 删除本地标签

```bash
git tag -d v1.0.0
#      -d：delete，删除本地标签（不影响远端）
```

### 删除远端标签

```bash
# 写法一（Git 1.7.0+）
git push origin --delete v1.0.0
#               --delete：删除远端的指定引用（分支或标签都适用）

# 写法二（推送一个空引用覆盖，效果等同于删除）
git push origin :refs/tags/v1.0.0
#              :<ref>：冒号前为空表示"推送空内容"，即删除远端的这个引用
```

两种写法效果一样，推荐第一种，语义更清晰。

---

## 检出标签：detached HEAD 的坑

```bash
git checkout v1.0.0
```

执行后会进入 **detached HEAD** 状态，HEAD 直接指向一个 commit 而不是分支。这时候如果做了新的提交，切走以后提交会丢失（没有分支引用它）。

如果只是查看历史代码，detached HEAD 没问题。但如果要在某个版本基础上修复 bug，正确做法是从标签创建一个分支：

```bash
git checkout -b hotfix/v1.0.1 v1.0.0
#            -b：新建并立即切换到这个分支（branch）
#               hotfix/v1.0.1：新分支名
#                              v1.0.0：起点，基于这个标签指向的 commit 创建分支
```

这样在 `v1.0.0` 的代码基础上，工作在 `hotfix/v1.0.1` 分支，提交不会丢。

---

## 常用命令速查

| 命令 | 说明 |
|------|------|
| `git tag <name>` | 创建轻量标签（当前 commit） |
| `git tag -a <name> -m "<msg>"` | 创建附注标签 |
| `git tag -a <name> <hash> -m "<msg>"` | 给历史 commit 打附注标签 |
| `git tag` | 列出所有本地标签 |
| `git tag -l "<pattern>"` | 通配符过滤标签列表 |
| `git show <name>` | 查看标签详情 |
| `git rev-parse <name>` | 查看标签指向的 commit hash |
| `git push origin <name>` | 推送单个标签 |
| `git push origin --follow-tags` | 推送所有附注标签 |
| `git push origin --tags` | 推送所有标签（含轻量标签） |
| `git tag -d <name>` | 删除本地标签 |
| `git push origin --delete <name>` | 删除远端标签 |
| `git checkout -b <branch> <name>` | 从标签创建新分支 |
| `git describe --tags` | 显示距最近 tag 的距离 |

---

## 版本发布的完整流程

以 `v1.2.0` 为例，典型流程：

```bash
# 1. 确保代码在主分支，工作区干净
git checkout main
git status

# 2. 打附注标签
git tag -a v1.2.0 -m "Release v1.2.0

主要变更：
- 新增 OAuth 登录支持
- 修复文件上传超时问题"

# 3. 推送代码和标签
git push origin main
git push origin v1.2.0

# 或者用 --follow-tags 合并成一步
git push origin main --follow-tags
```

推送后，GitHub 会在 Releases 页面列出这个标签，可以在页面上补充 Release Notes，也可以附加编译好的二进制文件。

### 语义版本命名（semver）

tag 名字没有强制要求，但遵循 [semver](https://semver.org/) 是行业惯例：

```
v<主版本>.<次版本>.<补丁版本>
v1.2.0   # 正式版本
v1.2.0-beta.1  # 预发布版本
v1.2.0-rc.1    # Release Candidate
```

- **主版本**：不兼容的 API 变更
- **次版本**：向下兼容的新功能
- **补丁版本**：向下兼容的 bug 修复

---

## `git describe`：距上一个 tag 有多远

```bash
git describe --tags
#            --tags：同时识别轻量标签，默认只识别附注标签
# v1.2.0-3-ga3f2c1d
```

输出格式：`<最近tag>-<距离commit数>-g<当前commit hash>`。在 CI 脚本里用来自动生成构建版本号很方便。如果当前 commit 刚好就是 tag，直接输出 `v1.2.0`。

---

## 常见问题

**git tag 和分支有什么区别？**

分支是移动的指针，随着新 commit 向前推进。tag 是固定的指针，一旦打上就不会移动，始终指向那个 commit。它们都是对 commit 的引用，但语义不同：分支表示"正在进行的工作线"，tag 表示"这个时刻的快照"。

**为什么 `git push` 不会自动推送标签？**

Git 把标签视为本地元数据，和分支一样不自动同步，需要显式推送。设计上是为了让临时的本地标签不污染远端。推送单个用 `git push origin v1.0.0`，推送全部附注标签用 `git push origin --follow-tags`。

**如何修改已经推送的标签？**

标签不能直接编辑。标准做法是删除旧标签再重新打：

```bash
# 本地删除并重新打
git tag -d v1.0.0                        # -d：删除本地标签
git tag -a v1.0.0 -m "新的 message"     # 重新打附注标签

# 同步到远端
git push origin --delete v1.0.0          # 删除远端旧标签
git push origin v1.0.0                   # 推送新标签
```

对于已经公开发布的版本号，强烈不建议这样操作，因为其他人可能已经基于这个 tag 做了构建或发布。

**轻量标签和附注标签哪个适合版本发布？**

附注标签（`git tag -a`）。它存储了完整的元数据（打标签者、时间、message），可以用 GPG 签名验证，`git describe` 默认只识别附注标签，`git show` 也能看到详细信息。轻量标签只适合本地临时标记，不适合正式版本。
