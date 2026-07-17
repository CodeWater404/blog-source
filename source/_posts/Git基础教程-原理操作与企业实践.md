---
title: "Git 基础教程：原理、常见操作与企业实践规范"
date: 2026-07-05 10:00:00
updated: 2026-07-17 16:14:35
cover: /img/p23.jpg
categories: git
tags:
  - Git
  - 版本控制
  - 工程实践
description: "Git 工作原理（三区域+对象模型）、常见命令速查、Git Flow 与 GitHub Flow 分支策略，以及企业中 commit message 规范、分支命名、PR 审查和 reflog 误操作恢复的完整实践指南。"
---

Git 是目前最主流的分布式版本控制系统，几乎所有软件团队都在用。但很多人停留在 `add / commit / push` 三板斧，遇到冲突或误操作就不知道怎么处理。这篇从原理出发，把常见操作和企业实践规范串起来。

<!-- more -->

---

## Git 的三个区域和分布式原理

### 工作区、暂存区、本地仓库

Git 把文件状态分成三个区域，理解这个是所有操作的基础：

```
工作区（Working Directory）
    ↓  git add
暂存区（Staging Area / Index）
    ↓  git commit
本地仓库（Local Repository）
    ↓  git push
远程仓库（Remote Repository）
```

- **工作区**：你实际编辑文件的目录
- **暂存区**：`git add` 后文件进入这里，是一个"预提交缓冲区"，可以精确控制这次 commit 包含哪些改动
- **本地仓库**：`git commit` 后改动被永久记录，存在 `.git/objects/` 目录
- **远程仓库**：`git push` 后同步到 GitHub/GitLab 等远端

暂存区是很多人忽略的设计：它让你可以把一个大改动拆分成多个有意义的 commit，而不是把所有修改塞进一个 commit。

### 分布式的含义

Git 是**分布式**的，每个开发者本地都有完整的仓库历史，不依赖中央服务器就能查历史、切分支、提交。这和 SVN 等集中式系统不同——SVN 断网就不能提交。

`git clone` 会把远端的完整历史都拉下来，所以 `git log` 是本地操作，不需要网络。

### Git 对象模型

Git 内部用四种对象存储所有内容，每个对象用其 SHA-1 哈希值唯一标识：

| 对象类型 | 存储内容 |
|---------|---------|
| **blob** | 文件内容（不含文件名） |
| **tree** | 目录结构（文件名 + blob 引用） |
| **commit** | 提交信息 + tree 引用 + parent commit 引用 |
| **tag** | 附注标签（指向 commit） |

一个 commit 对象大概长这样：

```
tree 8f3b2a...       # 指向这次提交的目录快照
parent 9c1d4f...     # 上一个 commit
author Dylan <...> 1720137600 +0800
committer Dylan <...> 1720137600 +0800

feat: add user login API
```

**branch 和 HEAD 是什么**：branch 只是一个指向某个 commit 的指针（存在 `.git/refs/heads/` 里），HEAD 是当前所在分支的指针。切换分支不是"复制文件"，只是移动指针，速度极快。

---

## 常见操作速查

### 初始化和克隆

```bash
# 初始化新仓库
git init

# 克隆远端仓库（含完整历史）
git clone https://github.com/user/repo.git

# 克隆指定分支（只拉这一个分支）
git clone -b main --single-branch https://github.com/user/repo.git
```

### 查看状态和差异

```bash
# 查看工作区和暂存区的状态
git status

# 查看工作区 vs 暂存区的差异（还没 add 的改动）
git diff

# 查看暂存区 vs 上次 commit 的差异（已 add 还没 commit 的改动）
git diff --staged

# 查看提交历史（图形化显示分支关系）
git log --oneline --graph --all
```

### 查看历史与追溯：谁改的、什么时候改的

排查"这段代码为什么长这样"时的三件套——看文件的修改历史、看某一行是谁改的、看某次提交到底改了什么：

```bash
# 查看某个文件的历史修改记录
git log --oneline src/main.go
#     只列出改过这个文件的提交，一行一条

git log -p src/main.go
#     -p：每条提交连带显示这个文件的具体 diff，改了什么一目了然

# 查看某一行（或某几行）是谁在哪次提交里改的
git blame src/main.go
#     逐行标注最后一次修改这一行的提交、作者和时间

git blame -L 10,20 src/main.go
#     -L：只看第 10 到 20 行，文件很长时不用翻

# 查看某个 commit 的完整内容
git show abc1234
#     显示这次提交的说明、作者、时间和完整 diff——
#     从 blame 或 log 里拿到可疑的 commit 哈希后，用它看细节
```

三个命令正好串成一条排查链：`git log 文件` 找到相关提交 → `git blame` 定位到具体某一行是哪次提交改的 → `git show` 看那次提交的完整改动和说明。

> 更进一步——想看某个函数或某段代码块（而不是整个文件）的完整历史，用 `git log -L`，踩坑细节见：[Git: 查看指定代码块的历史记录](/Git-view-the-history-of-the-specified-code-block)

### add 和 commit

```bash
# 把指定文件加入暂存区
git add src/main.go

# 把所有改动加入暂存区（新增、修改、删除）
git add .

# 交互式选择要暂存的代码块（可以只提交文件的一部分）
git add -p src/main.go

# 提交暂存区的内容
git commit -m "feat: add user login API"

# 跳过暂存区，直接提交所有已追踪文件的修改（不含新文件）
git commit -am "fix: correct status code"
```

`git add -p` 是个容易被忽视的功能：当一个文件有多处修改，但只想把其中一部分提交时，用它可以逐块选择。

### 分支操作

```bash
# 查看本地分支
git branch

# 查看所有分支（含远端）
git branch -a

# 创建并切换到新分支
git switch -c feature/login
# 等价于旧写法：git checkout -b feature/login

# 切换到已有分支
git switch main

# 删除本地分支（-d 只删已合并的，-D 强制删除）
git branch -d feature/login

# 删除远端分支
git push origin --delete feature/login
```

### 合并和变基

```bash
# 合并分支（在 main 分支上执行，把 feature/login 合并进来）
git merge feature/login

# 合并并强制生成一个 merge commit（即使可以 fast-forward）
git merge --no-ff feature/login

# 变基（在 feature 分支上执行，把 feature 的提交移到 main 最新节点之后）
git rebase main
```

**merge vs rebase 的区别**：
- `merge` 保留完整历史，会生成一个 merge commit，清楚地记录"这两条线在这里合并了"
- `rebase` 重写 feature 分支的提交历史，让它看起来像是在最新的 main 上线性开发的，历史更干净，但会改变 commit 的 SHA

> **已推送到远端的分支不要 rebase**，rebase 会改变提交历史，会和别人的本地仓库产生冲突。

### 远端操作

```bash
# 查看远端仓库配置
git remote -v

# 拉取远端改动并合并（= fetch + merge）
git pull

# 拉取远端改动并变基（保持线性历史）
git pull --rebase

# 推送到远端
git push origin feature/login

# 第一次推送时设置上游分支（之后 git push 不需要带参数）
git push -u origin feature/login
```

### 暂存工作区：stash

`git stash` 把当前工作区和暂存区的改动临时存起来，让工作区恢复干净状态，适合"临时切换分支处理紧急 bug"的场景。

```bash
# 把当前改动暂存起来
git stash

# 暂存时加说明
git stash push -m "WIP: login form validation"

# 查看所有 stash
git stash list

# 恢复最近一次 stash（保留 stash 记录）
git stash apply

# 恢复并删除最近一次 stash 记录
git stash pop

# 恢复指定的 stash
git stash apply stash@{2}

# 查看某个 stash 改动了哪些文件
git stash show stash@{0}

# 查看某个 stash 的完整 diff（加 -p）
git stash show stash@{0} -p
```

### 回滚操作

```bash
# 撤销工作区的修改（恢复到上次 commit 的状态，改动丢失）
git restore src/main.go

# 交互式部分还原：逐块询问，选择性地撤销文件里的一部分修改
git restore -p src/main.go

# 把文件从暂存区移回工作区（取消 add）
git restore --staged src/main.go

# 回退到上一个 commit，保留工作区改动（最常用）
git reset HEAD~1

# 回退到指定 commit，保留工作区改动
git reset abc1234

# 回退到上一个 commit，同时清除工作区改动（危险，改动会丢失）
git reset --hard HEAD~1

# 回退到指定 commit，同时清除工作区改动（危险）
git reset --hard abc1234

# 生成一个新的 commit 来撤销某次提交（不改历史，适合已推送的情况）
git revert abc1234
```

`reset --hard` 会直接丢弃工作区改动，执行前确认好。已经 push 的提交用 `revert` 更安全，它不改历史，而是新增一个撤销的 commit。

`restore -p` 单独展开讲一下，它解决的是一个很常见的场景：一个文件里混着有效修改和无效修改——比如改了几行逻辑，编辑器顺手又格式化出一堆空格变动、删掉的空行，`git diff` 一看满屏噪音。整个文件 `restore` 会把有效修改一起丢掉，手动改回去又太费劲。`-p`（patch）模式会把文件的改动拆成一个个块（hunk），逐块展示 diff 并询问怎么处理：

```bash
git restore -p src/main.go
#     对每一块改动，git 会显示 diff 并问 Discard this hunk from worktree?
#     y：还原这一块（丢弃这块改动）
#     n：保留这一块，看下一块
#     s：这一块太大，拆成更小的块再逐个问
#     q：退出，后面的块全部保留
```

遇到"空格、空行这类无效修改混在有效修改里"的情况，逐块 y/n 过一遍，噪音全清掉、有效改动一行不丢。同样的 `-p` 参数在 `git add -p` 上也能用，方向反过来——从混杂的改动里挑出想提交的部分。

---

## 分支策略：Git Flow 与 GitHub Flow

多人协作时，如果每个人随意创建分支、随意往 main 合并，很快就会出现以下问题：main 上有未测试的代码导致线上崩溃；不知道哪个分支是"当前最新"的；hotfix 合到了错误的地方，修复没生效。

**分支策略**就是团队约定好"什么分支做什么事、从哪切出、合到哪里去"，把协作流程标准化，避免上面这些混乱。不同规模和发布节奏的团队适合不同的策略，没有统一答案，但必须有一套。

### Git Flow

Git Flow 是经典的企业分支模型，有两条长期分支和三类临时分支：

**长期分支：**
- `main`（或 `master`）：只存放生产环境的稳定代码，每次更新都打 tag
- `develop`：集成分支，所有功能开发完成后先合并到这里

**临时分支（用完即删）：**
- `feature/xxx`：从 develop 切出，功能开发完合回 develop
- `release/x.x.x`：从 develop 切出，做最终测试和 bug 修复，完成后合并到 main 和 develop
- `hotfix/xxx`：从 main 切出，紧急修复生产 bug，完成后合并到 main 和 develop

**适合场景**：有明确版本节奏的项目（比如定期发版的 App、SDK）。

**缺点**：分支多、流程重，小团队或持续交付场景下嫌麻烦。

### GitHub Flow

GitHub Flow 更轻量，只有一条规则：**main 随时可部署，所有改动通过 PR 合并**。

```
main（始终可部署）
  ↑ PR + Code Review
feature/xxx（从 main 切出，完成后 PR 合回）
```

流程：
1. 从 main 创建 feature 分支
2. 在 feature 分支开发、提交
3. 发起 PR，Code Review
4. CI 通过后合并到 main
5. 立即部署

**适合场景**：持续交付、小团队、Web 服务（随时上线）。

### 怎么选

| 场景 | 推荐 |
|------|------|
| 有固定发版周期（App、SDK） | Git Flow |
| 持续部署、随时上线 | GitHub Flow |
| 小团队、快速迭代 | GitHub Flow |
| 需要维护多个版本 | Git Flow |

---

## 企业实践规范

### commit message 规范：Conventional Commits

混乱的 commit message 让 `git log` 毫无意义，也没法自动生成 changelog。[Conventional Commits](https://www.conventionalcommits.org/) 是目前最主流的规范：

```
<type>(<scope>): <subject>

[可选 body]

[可选 footer]
```

常用 type：

| type | 含义 |
|------|------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `docs` | 文档改动 |
| `refactor` | 重构（不影响功能） |
| `test` | 测试相关 |
| `chore` | 构建、依赖、CI 等杂项 |
| `perf` | 性能优化 |
| `revert` | 撤销某次提交 |

示例：

```
feat(auth): add Google OAuth login

Support sign-in with Google account via OAuth 2.0.
Closes #42

fix(api): return 404 instead of 500 for missing user

chore: upgrade Go version to 1.22
```

**实践建议**：
- subject 用英文或中文均可，保持团队统一就行
- subject 不超过 72 个字符
- 用祈使句（"add"，不是"added"或"adding"）
- 破坏性改动在 footer 加 `BREAKING CHANGE: 说明`

可以用 [commitlint](https://commitlint.js.org/) 在 git hook 里强制校验格式。

### 分支命名规范

```
feature/issue-42-user-login      # 新功能（关联 issue 号）
fix/issue-88-null-pointer        # bug 修复
hotfix/payment-timeout           # 紧急修复
release/v1.2.0                   # 发布分支
chore/upgrade-dependencies       # 杂项
```

关联 issue 号的好处：在 GitHub/GitLab 上可以直接从分支名跳转到对应 issue，review 时方便查背景。

### 保护主分支

在 GitHub/GitLab 上给 `main` 开启 branch protection：
- **禁止直接 push**，所有改动必须通过 PR
- **PR 合并前必须通过 CI**（单测、lint、构建）
- **至少一个 Code Review 通过**才能合并
- **禁止 force push**

这几条规则能防止 90% 的低级事故。

### PR/MR 规范

一个好的 PR 应该：
- **只做一件事**：一个 PR 只解决一个问题，越小越容易 review
- **有清晰的描述**：说明改了什么、为什么改、怎么测试
- **关联 issue**：在描述里加 `Closes #42`，合并后自动关闭 issue
- **自己先 review 一遍**：提交前 diff 看一眼，去掉调试代码和无关改动

### .gitignore 配置

`.gitignore` 要在项目初始化时就配好，不要等到不该提交的文件已经进了仓库再处理。

常见需要忽略的内容：

```gitignore
# 依赖
node_modules/
vendor/

# 构建产物
dist/
build/
*.class
*.o

# 环境配置（含密钥，绝对不能提交）
.env
.env.local
*.pem
*.key

# IDE
.idea/
.vscode/
*.swp

# 系统文件
.DS_Store
Thumbs.db
```

[gitignore.io](https://www.toptal.com/developers/gitignore) 可以按语言和框架自动生成。

---

## 常见问题处理

### 误操作恢复：git reflog

`git reflog` 记录了 HEAD 的所有移动历史，包括 `reset --hard` 这类"危险操作"之后的状态。只要没有 `git gc`，几乎所有误操作都能恢复。

```bash
# 查看 HEAD 的操作历史
git reflog

# 输出示例：
# abc1234 HEAD@{0}: reset: moving to HEAD~1
# def5678 HEAD@{1}: commit: feat: add login page
# ghi9012 HEAD@{2}: commit: fix: correct redirect URL

# 恢复到 reset 之前的状态（def5678 是 reset 之前的 commit）
git reset --hard def5678
```

`reset --hard` 把工作区搞没了？先跑 `git reflog`，找到那个 commit 的哈希，`reset --hard` 回去就行。

### 解决合并冲突

冲突发生时 Git 会在文件里标记冲突区域：

```
<<<<<<< HEAD
你的改动（当前分支）
=======
对方的改动（被合并的分支）
>>>>>>> feature/login
```

处理步骤：
1. 用编辑器打开冲突文件，选择保留哪一方或手动合并
2. 删除所有 `<<<<<<<`、`=======`、`>>>>>>>` 标记
3. `git add` 标记冲突已解决
4. `git commit` 完成合并

```bash
# 查看哪些文件有冲突
git status

# 解决后标记为已解决
git add src/main.go

# 完成合并提交
git commit
```

如果冲突太复杂，可以用 `git mergetool` 调出可视化合并工具（需要先配置）。

改到一半发现冲突超出预期、想放弃这次合并重新来过，不用手动去删那些冲突标记——直接取消合并，回到合并前的状态：

```bash
git merge --abort
#     取消这次合并，工作区恢复到 merge 之前的样子，就像什么都没发生过

git rebase --abort
#     变基过程中遇到冲突想放弃，对应的取消命令是这个
```

注意前提：合并前的工作区应该是干净的（没有未提交的改动）——这也是为什么合并前先 commit 或 stash 是个好习惯，否则 `--abort` 恢复现场时可能没法完整还原你未提交的那部分内容。

### 修改最近一次 commit

```bash
# 修改最近一次 commit 的 message
git commit --amend -m "feat: add Google OAuth login"

# 把忘加的文件补进最近一次 commit（不改 message）
git add forgotten_file.go
git commit --amend --no-edit
```

`--amend` 会重写最近一次 commit，产生新的 SHA。**已推送到远端的 commit 不要 amend**，否则需要 force push，会影响其他人。

### 修改指定 commit：git rebase -i

`--amend` 只能改最近一次 commit，修改更早的提交要用交互式 rebase。

```bash
# 修改最近 3 个 commit（打开交互式编辑界面）
git rebase -i HEAD~3
```

弹出的编辑器里每行是一个 commit，把要修改的那行前面的 `pick` 改成对应指令：

| 指令 | 含义 |
|------|------|
| `pick` | 保留这个 commit（默认） |
| `reword` | 只修改 commit message |
| `edit` | 暂停在这个 commit，可以修改内容 |
| `squash` | 把这个 commit 合并到上一个 commit |
| `drop` | 删除这个 commit |

以修改 message 为例，把 `pick` 改成 `reword`，保存退出，Git 会再弹出编辑器让你改 message。

以修改内容为例，把 `pick` 改成 `edit`，保存退出后 Git 会暂停在那个 commit：

```bash
# Git 提示你现在在指定 commit，可以修改文件
# 改完后：
git add src/main.go
git commit --amend --no-edit   # 把改动追加到当前 commit
git rebase --continue          # 继续完成剩余的 rebase
```

> **同样：已推送到远端的 commit 不要用 rebase -i 修改**，会改写历史，影响所有人。

### cherry-pick：把某个 commit 单独移植

```bash
# 把 abc1234 这个 commit 的改动应用到当前分支
git cherry-pick abc1234

# 移植多个连续的 commit（包含 abc1234 到 def5678，注意要加 ^）
git cherry-pick abc1234^..def5678
```

场景：hotfix 在 main 分支修复了一个 bug，需要把这个修复也应用到正在维护的 release 分支，用 cherry-pick 比重新写一遍高效。
