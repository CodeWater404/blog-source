---
title: "终端搜索三件套：fzf + ripgrep + fd，以及 jq"
date: 2026-06-30 14:00:00
updated: 2026-07-04 14:10:00
cover: /img/p17.jpg
categories: tools
tags:
  - Mac
  - CLI
  - Terminal
  - fzf
  - ripgrep
  - fd
  - jq
description: "用 fzf、ripgrep、fd、jq 替换 Mac 终端里的 grep/find：模糊搜索框架让任何命令可交互，rg 在大型仓库里秒级全文搜索，fd 语法比 find 直觉十倍，jq 处理 API 返回的 JSON。"
---

`grep -r` 在大型仓库里要等好几秒，`find` 的参数从来记不住，API 返回的 JSON 只能整块看。fzf、ripgrep、fd 和 jq 把这三个问题解决了，而且可以串联成一条搜索流水线。

这篇是系列第二篇，接在 [kitty + Zim](/Mac终端基础-kitty和Zim) 之后。如果你在 Zim 里启用了 `fzf-tab` 插件，装完 fzf 之后它会自动生效。

<!-- more -->

---

## [fzf](https://github.com/junegunn/fzf)：模糊搜索框架

fzf 本身做一件事：接受文本输入，让你模糊过滤后选出一行。但它能接在任何命令后面，这让它成为整个工具链里"乘数效应"最强的一个。

默认行为：从 stdin 读取文本列表，输出用户选中的行到 stdout；单独运行时调用 `find` 或 `$FZF_DEFAULT_COMMAND` 列出文件；模糊匹配不要求字符连续，只要按顺序出现即可（输入 `gc` 能匹配 `git commit`）。

### 安装与 Zsh 集成

```bash
brew install fzf
# 安装 Zsh 快捷键绑定——不跑这条，三个快捷键不会自动生效
$(brew --prefix)/opt/fzf/install
```

装完重启终端（或 `exec zsh`），三个快捷键立即可用。

### 三个核心快捷键

**`Ctrl+T`：模糊查找文件，路径粘到命令行**

```bash
# 你想 nvim 打开某个文件，但记不清路径
nvim <按 Ctrl+T>
# 弹出 fzf 界面，模糊搜索当前目录下所有文件，选中后路径自动补全到命令行
```

**`Alt+C`：模糊跳转目录**

```bash
# 按 Alt+C 弹出目录列表，模糊搜索后直接 cd 过去
# 比输 cd ~/code/github/some-project/src/components 快得多
```

> kitty 用户注意：需要 `kitty.conf` 里有 `macos_option_as_alt yes`，Alt+C 才能触发。第一篇里已经配过了。

**`Ctrl+R`：交互式历史搜索**

替换原生的 `Ctrl+R`，弹出可模糊搜索的历史列表，选中直接执行。如果你后续装了 atuin，atuin 会接管这个快捷键，功能更强（第五篇讲）。

### fzf-tab 在 Zim 里自动生效

第一篇 Zim 配置里加了 `fzf-tab` 插件。只要 fzf 装好，重启终端后按 `Tab` 补全就会变成 fzf 交互界面——候选列表可以模糊过滤，用 `↑↓` 选择，回车确认。

### 常用参数

| 参数 | 说明 |
|------|------|
| `-m` | 多选（Tab 标记，回车一次输出所有选中项） |
| `-e` | 精确匹配，不模糊（适合搜确切文件名） |
| `-1` | 只有一个匹配时自动选择，不弹界面 |
| `-0` | 没有匹配时直接退出，不弹界面 |
| `--query "str"` | 启动时预填搜索词 |
| `--header "text"` | 在列表顶部显示说明文字 |
| `--prompt "str"` | 自定义提示符（默认 `> `） |
| `--preview 'cmd'` | 预览命令，`{}` 代表当前高亮项 |
| `--preview-window pos:size` | 预览窗口位置（`right`/`left`/`up`/`down`）和大小（如 `right:50%`） |
| `--bind "key:action"` | 自定义按键绑定（如 `ctrl-y:execute(echo {} \| pbcopy)`） |
| `--filter "str"` | 非交互模式，直接过滤输出，不弹界面 |
| `--nth N` | 只对第 N 列做模糊匹配（配合 `--delimiter` 用） |
| `--with-nth N` | 只显示第 N 列，不影响实际输出内容 |
| `--no-sort` | 保持输入顺序，不按匹配度排序 |
| `--tac` | 反转输入顺序（常用于让最新历史显示在上方） |

几个实用示例：

```bash
# 只有一个匹配时自动选择，避免弹出只有一项的界面
git branch | fzf -1 | xargs git checkout

# 带说明文字的选择界面
printf "yes\nno" | fzf --header "确认删除？" --prompt "选择: "

# 预览窗口显示在下方，占 40% 高度
fzf --preview 'cat {}' --preview-window down:40%

# 对 ps aux 输出只匹配第 11 列（进程名），避免 PID 干扰匹配
ps aux | fzf --nth 11

# 非交互过滤：不弹界面，直接输出匹配结果（类似模糊版 grep）
cat list.txt | fzf --filter "keyword" --no-sort

# 按 Ctrl+Y 复制当前选中项到剪贴板
fzf --bind 'ctrl-y:execute(echo {} | pbcopy)'
```

### 和其他命令组合

fzf 接受任意文本输入，输出用户选中的行：

```bash
# 交互式切换 git 分支
git branch | fzf | xargs git checkout

# 交互式 kill 进程
ps aux | fzf | awk '{print $2}' | xargs kill

# 带预览窗口的文件浏览（需要先装 bat，第三篇会讲）
fzf --preview 'bat --color=always {}'
```

### 多选：-m

加 `-m` 参数后，Tab 键可以多选，回车一次性输出所有选中项：

```bash
# 同时用 nvim 打开多个文件
fd -t f | fzf -m | xargs nvim

# 同时删除多个匹配文件（慎用）
fd -e log | fzf -m | xargs rm
```

### 实时全文搜索：把 rg 接进 fzf

下面这个函数可以加到 `~/.zshrc` 里，实现"边输边搜"的交互式全文搜索：

```bash
# 加到 ~/.zshrc
rg-fzf() {
  local RG_PREFIX="rg --column --line-number --no-heading --color=always --smart-case"
  fzf --ansi --disabled --query "$*" \
    --bind "start:reload:$RG_PREFIX {q}" \
    --bind "change:reload:sleep 0.1; $RG_PREFIX {q} || true" \
    --bind "enter:become(nvim {1} +{2})" \
    --delimiter : \
    --preview 'bat --color=always {1} --highlight-line {2}' \
    --preview-window 'right,60%,border-left,+{2}+3/3,~3'
}
```

调用方式：`rg-fzf "关键词"` 或直接 `rg-fzf`，在 fzf 界面里继续输入过滤。输入关键词时 rg 实时重新查询，选中条目后按回车用 nvim 打开对应文件并跳到对应行。

### FZF_DEFAULT_OPTS：统一配置预览窗口

在 `~/.zshrc` 里设置 `FZF_DEFAULT_OPTS`，可以让所有 fzf 调用都带上默认选项：

```bash
# 加到 ~/.zshrc
export FZF_DEFAULT_OPTS="
  --height=60%
  --layout=reverse
  --border=rounded
  --preview-window=right:50%
"
```

设置之后，单独运行 `fzf` 也会默认带预览窗口布局，不需要每次手动传参。

---

## [ripgrep](https://github.com/BurntSushi/ripgrep)（rg）：比 grep 快一个数量级的全文搜索

ripgrep（rg）是用 Rust 编写的全文搜索工具。默认行为：第一个参数当正则表达式，无需 `-r` 就递归搜索当前目录；自动跳过 `.git/`、`.gitignore` 里的路径和隐藏文件；输出带文件名、行号和彩色高亮。速度比 `grep -r` 快一个数量级，主要原因是多线程并发 + 跳过大量无关文件。

### 安装

```bash
brew install ripgrep
```

### 核心用法

```bash
# 在当前目录递归搜索，自动跳过 .gitignore 里的内容
rg "pattern"

# 搜索特定类型文件
rg -t go "http.Handle"
rg -t py "import requests"

# 不区分大小写
rg -i "TODO"

# 整词匹配（不匹配 patterns 里的 pattern）
rg -w "error"

# 在指定目录下搜索
rg "pattern" src/
rg "pattern" ~/code/github/some-project

# 只输出匹配的文件名
rg -l "pattern"

# 显示上下文（前后各 2 行）
rg -C 2 "panic"
```

`rg` 和 `grep -r` 的主要区别：不需要加 `-r` 就默认递归；自动跳过 `.git/`、`.gitignore` 里的路径、隐藏文件；输出带文件名、行号和彩色高亮。在大型代码仓库里，速度差距非常明显。

### 常用参数

| 参数 | 说明 |
|------|------|
| `-i` | 忽略大小写 |
| `-s` | 强制区分大小写 |
| `-S` / `--smart-case` | 智能大小写：有大写字母时区分，全小写时忽略 |
| `-w` | 整词匹配 |
| `-F` | 固定字符串（不当作正则，特殊字符不需要转义） |
| `-v` | 反向匹配（输出不包含 pattern 的行） |
| `-l` | 只输出文件名，不显示匹配行 |
| `-c` | 只输出每个文件的匹配行数 |
| `-o` | 只输出匹配的部分，不输出整行 |
| `-n` / `-N` | 显示/隐藏行号（默认显示） |
| `-A N` | 匹配行后面 N 行 |
| `-B N` | 匹配行前面 N 行 |
| `-C N` | 匹配行前后各 N 行 |
| `-t type` | 只搜索指定类型（`go`/`py`/`js`/`md` 等） |
| `-T type` | 排除指定类型 |
| `-g "glob"` | 只搜索匹配 glob 的文件（如 `-g "*.go"`） |
| `--hidden` | 包含隐藏文件（默认跳过） |
| `-u` | 不跳过 `.gitignore` 里的文件 |
| `-m N` | 每个文件最多输出 N 条匹配 |
| `-e "pattern"` | 指定多个匹配模式（可多次用） |
| `--json` | 输出 JSON 格式（可接 jq 处理） |

```bash
# 在 .go 和 .proto 文件里搜索，忽略大小写，显示前后 3 行
rg -t go -g "*.proto" -i -C 3 "grpc"

# 只看匹配的部分（不是整行），适合提取变量名等
rg -o 'GOPATH=\S+' ~/.zshrc

# 反向匹配：找所有不含 TODO 的 Go 文件
rg -l --files-without-match "TODO" -t go

# 多个 pattern 任意匹配
rg -e "error" -e "panic" -e "fatal" main.go

# 输出 JSON 后用 jq 处理（行号、文件名等结构化取出）
rg --json "pattern" | jq 'select(.type=="match") | .data.path.text'
```

### 和 fzf 组合：交互式选择搜索结果

```bash
# 列出所有匹配的文件，fzf 里预览内容并选择
rg -l "TODO" | fzf --preview 'bat --color=always {}'

# 选中后直接用 nvim 打开
rg -l "TODO" | fzf --preview 'bat --color=always {}' | xargs nvim
```

### 全局配置：~/.ripgreprc

每次都手动传参数很麻烦。把常用选项写进配置文件，rg 会自动加载：

```bash
# 创建配置文件
touch ~/.ripgreprc
```

```
# ~/.ripgreprc 内容

# 有大写字母时区分大小写，全小写时忽略大小写——比 -i 更智能
--smart-case

# 搜索隐藏文件（如 .env、.zshrc），但仍跳过 .git/
--hidden
--glob=!.git/

# 输出带颜色（在管道里也保持颜色）
--color=always
```

然后在 `~/.zshrc` 里指定配置文件路径：

```bash
export RIPGREP_CONFIG_PATH="$HOME/.ripgreprc"
```

配置生效后，直接 `rg "pattern"` 就等于 `rg --smart-case --hidden "pattern"`，不需要每次手动传这些参数。

---

## [fd](https://github.com/sharkdp/fd)：find 的现代替代品

fd 是 `find` 的现代替代品，同样用 Rust 编写。默认行为：第一个参数当正则表达式匹配文件名（不是路径），大小写不敏感；自动跳过 `.gitignore` 里的路径和以 `.` 开头的隐藏文件；输出带颜色。不加任何参数时列出当前目录下所有非隐藏文件，等同于 `find . -not -path '*/.*'` 但更快。

### 安装

```bash
brew install fd
```

### 核心用法

fd 和 `find` 做同样的事，但语法直觉很多：

```bash
# 找文件名包含 config 的文件（默认递归，大小写不敏感）
fd config

# 按扩展名查找
fd -e go
fd -e md -e txt         # 多个扩展名

# 按类型过滤
fd -t f                 # 只找文件（file）
fd -t d src             # 只找目录（directory）

# 排除目录
fd -E node_modules -E .git -e js

# 在特定路径下查找
fd -e go ~/code/github

# 对结果执行命令（类似 find -exec）
fd -e go -x wc -l       # 统计每个 Go 文件行数（-x 逐个执行）
fd -e log -X rm         # 删除所有 .log 文件（-X 合并参数）
```

`fd` vs `find` 语法对比：

```bash
# 找当前目录下所有 .go 文件
find . -name "*.go" -type f   # find
fd -e go                       # fd

# 找名字包含 test 的文件
find . -name "*test*" -type f  # find
fd test                        # fd
```

### 常用参数

| 参数 | 说明 |
|------|------|
| `-e ext` | 按扩展名过滤（不带点，可多次用） |
| `-t f/d/l/x` | 按类型：文件/目录/符号链接/可执行文件 |
| `-E pattern` | 排除匹配的路径（可多次用） |
| `-H` | 包含隐藏文件（以 `.` 开头的） |
| `-I` | 不跳过 `.gitignore`（忽略所有 ignore 规则） |
| `-L` | 跟随符号链接 |
| `-d N` | 最大搜索深度（`-d 1` 只搜当前目录） |
| `-g` | 使用 glob 模式而不是正则（`-g "*.go"` 等） |
| `-s` | 区分大小写（默认不区分） |
| `-l` | 显示详细信息（类似 `ls -l`） |
| `-0` | 用 null 分隔输出（配合 `xargs -0` 处理含空格的路径） |
| `-x cmd` | 对每个结果执行命令（逐个） |
| `-X cmd` | 对所有结果合并执行一次命令 |
| `--changed-within N` | 在 N 时间内修改过的（如 `1h`、`2d`、`1week`） |
| `--changed-before N` | 在 N 时间之前修改过的 |
| `--max-results N` | 最多输出 N 条结果 |

```bash
# 找最近 1 小时内改动过的文件
fd --changed-within 1h

# 找 7 天前的日志文件（可以接 -X rm 清理）
fd -e log --changed-before 7d

# 搜索包含空格的路径时，用 -0 + xargs -0 避免分词问题
fd -e txt -0 | xargs -0 wc -l

# 只搜当前目录一层（不递归进子目录）
fd -d 1

# 找所有可执行文件（排除 .git 目录）
fd -t x -E .git

# 找隐藏配置文件（如 .env、.zshrc）
fd -H -g ".*rc" ~
```

### 和 fzf 组合

```bash
# 列出所有文件，fzf 选择后用 nvim 打开
fd -t f | fzf | xargs nvim

# 列出所有目录，fzf 选择后 cd 进去
cd $(fd -t d | fzf)
```

---

## [jq](https://jqlang.github.io/jq/)：JSON 命令行处理器

jq 是 JSON 的命令行处理器，作用类似 `sed`/`awk`，但专门处理 JSON 结构。默认行为：从 stdin 读取 JSON，第一个参数是过滤表达式（`.` 表示原样输出），输出默认 pretty-print（带缩进和颜色高亮）；字符串输出带双引号——新手最常踩的坑，加 `-r` 才能去掉引号，把结果传给 shell 用。

### 安装

```bash
brew install jq
```

### 基础语法

jq 的查询表达式作为第一个参数传入：

```bash
# 格式化输出（pretty-print）
cat data.json | jq .

# 提取字段
echo '{"name": "Alice", "age": 30}' | jq '.name'
# 输出: "Alice"

# 嵌套字段
jq '.data.user.email'

# 遍历数组
jq '.[]'           # 展开每个元素
jq '.[0]'          # 第一个元素
jq '.[-1]'         # 最后一个元素

# 提取数组里每个元素的字段
jq '.[].name'
jq '.items[] | .id'

# 过滤
jq '.[] | select(.status == "active")'
jq '.[] | select(.age > 18)'

# 重组输出（只保留指定字段）
jq '.[] | {id, name}'
jq '.[] | {user: .name, email: .contact.email}'
```

`-r` 参数去掉字符串的引号，适合把结果传给其他命令：

```bash
jq -r '.name'       # 输出 Alice，而不是 "Alice"
```

### 常用参数

| 参数 | 说明 |
|------|------|
| `-r` | 原始输出：字符串不加引号（适合传给 shell） |
| `-c` | 紧凑输出：不格式化，单行输出（适合写入文件或传给程序） |
| `-n` | 不读取输入，从 `null` 开始（用于构造 JSON） |
| `-e` | 输出为 `false`/`null` 时以非零退出码退出（脚本判断用） |
| `-s` | 把所有输入合并成一个数组再处理 |
| `-R` | 把每行输入当作原始字符串（不解析 JSON） |
| `-j` | 等同 `-r`，输出后不加换行 |
| `-S` | 对象的键按字母排序输出 |
| `--arg name val` | 把 shell 变量作为字符串传入 jq（在表达式里用 `$name`） |
| `--argjson name val` | 把 shell 变量作为 JSON 值传入（可以是数字、数组等） |
| `--slurpfile name file` | 把 JSON 文件内容作为变量传入 |
| `--indent N` | 缩进 N 个空格（默认 2） |
| `--tab` | 用 Tab 缩进 |

```bash
# -c：单行输出，适合写日志或传给程序
jq -c '.[]' data.json

# -n：不读输入，直接构造 JSON
jq -n '{name: "Alice", scores: [90, 85, 92]}'

# -s：把多个 JSON 文件合并成数组后处理
jq -s '.[0].users + .[1].users' a.json b.json

# -R + -s：把普通文本文件转成 JSON 字符串数组（每行一个元素）
cat list.txt | jq -Rs 'split("\n") | map(select(. != ""))'

# -e：在脚本里判断字段是否存在
if jq -e '.error' response.json > /dev/null 2>&1; then
  echo "请求失败"
fi

# --argjson：传入数字（不用 --arg，--arg 传的都是字符串）
MIN=100
jq --argjson min "$MIN" '.[] | select(.count > $min)' data.json
```

### 实际场景：curl | jq

```bash
# 查看 GitHub 用户信息，只取关键字段
curl -s https://api.github.com/users/octocat | jq '{name, public_repos, followers}'

# 列出仓库名和 star 数
curl -s https://api.github.com/users/octocat/repos \
  | jq '.[] | {name, stargazers_count}'

# 只看 star 数大于 100 的仓库
curl -s https://api.github.com/users/octocat/repos \
  | jq '[.[] | select(.stargazers_count > 100) | {name, stargazers_count}]'

# 从本地 JSON 配置文件提取字段
jq '.database.host' config.json
```

### 进阶：排序、统计、变量传入

```bash
# 按 star 数排序（降序）
curl -s https://api.github.com/users/octocat/repos \
  | jq 'sort_by(-.stargazers_count) | .[] | {name, stargazers_count}'

# 统计数组长度
echo '[1,2,3,4]' | jq 'length'
# 输出: 4

# 获取对象所有键名
echo '{"name":"Alice","age":30}' | jq 'keys'
# 输出: ["age","name"]

# 检查字段是否存在
jq '.[] | select(has("email"))' users.json

# 传入 shell 变量（用 --arg，避免引号和注入问题）
STATUS="active"
jq --arg s "$STATUS" '.[] | select(.status == $s)' data.json
```

### 输出格式转换

```bash
# 输出 CSV（适合导入 Excel 或传给其他工具）
curl -s https://api.github.com/users/octocat/repos \
  | jq -r '.[] | [.name, .stargazers_count, .language] | @csv'
# 输出:
# "Hello-World",1234,"JavaScript"
# "Spoon-Knife",567,"HTML"

# 输出 TSV（制表符分隔）
jq -r '.[] | [.id, .name] | @tsv' data.json

# 提取字段列表，传给 shell 命令
# -r 去引号，结果可以直接 xargs 处理
curl -s https://api.github.com/users/octocat/repos \
  | jq -r '.[].clone_url' | head -5
```

`@csv` 和 `@tsv` 会自动处理字段里的引号和特殊字符，比手动拼字符串安全。

---

## 管道把四个工具串成工作流

四个工具单独用都够用，组合起来才是真正的效率提升。fzf 在中间当连接器，把 rg/fd 找到的结果变成交互式选择，bat 做文件预览：

```bash
# 在所有 Go 文件里搜索 TODO，fzf 选择后用 nvim 打开
rg -t go -l "TODO" | fzf --preview 'bat --color=always {}' | xargs nvim

# 找配置文件，预览内容，选择后编辑
fd -e json -e yaml | fzf --preview 'bat --color=always {}' | xargs nvim

# 找到包含某个函数名的文件，查看行号
rg -n "func HandleLogin" --type go
```

以上命令里的 `bat --color=always` 作为预览器，但 bat 还没装。下一篇讲 bat + eza + glow + mdcat，bat 装好后预览窗口就都能用了。
