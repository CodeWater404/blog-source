---
title: "npm 与 pnpm 入门：两个包管理器怎么用，区别在哪"
date: 2026-09-01 23:53:17
categories: tools
tags:
  - npm
  - pnpm
  - Node.js
  - 前端工程化
cover: /img/p73.jpg
description: "面向没用过包管理器的新手：npm 和 pnpm 分别是什么、怎么装包、常用命令怎么对应，以及它们装出来的东西为什么不一样。"
---

写 JavaScript 项目时，代码里经常会用到别人写好的代码库，比如一个处理时间的库、一个搭 Web 服务的框架。这些别人写好、可以直接拿来用的代码库，就叫**包（package）**。**包管理器**就是专门用来"下载包、记录用了哪些包、装进项目里"的工具。npm 和 pnpm 都是包管理器，装出来的结果能让你的代码正常跑起来，这一点上它们是等价的，区别在怎么装、装出来的东西长什么样。

本文所有命令和结果都在 Node v20.19.2、npm 11.4.0、pnpm 10.15.1 上实际跑过。

<!-- more -->

## npm 是什么：Node.js 自带的包管理器

npm 是 Node.js 官方自带的包管理器，装了 Node.js 就自带了 npm，不用额外安装。它的默认行为是：读项目里一个叫 `package.json` 的文件，把里面记录的包挨个下载下来，放进一个叫 `node_modules` 的文件夹里。

跟着做一遍就知道它具体在干什么。新建一个空文件夹当项目：

```bash
mkdir npm-demo && cd npm-demo

npm init -y
#         -y：跳过一路的问答（项目名、版本号之类），直接生成一份默认的 package.json
```

跑完这条命令，目录里多了一个 `package.json` 文件：

```json
{
  "name": "npm-demo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

暂时只用关心 `name` 和 `version`，别的字段（`description`、`scripts` 之类）是给项目本身用的元信息，跟包管理没直接关系，先不管。这个文件后面会随着装包越来越关键——它就是整个项目"用了哪些包"的清单。现在往项目里装一个真实的包，用 [express](https://expressjs.com/)（一个搭 Web 服务用的框架）举例：

```bash
npm install express
#   install：下载这个包，并且记进 package.json
```

跑完之后，`package.json` 里自动多了这么一段（其他字段没变，省略掉了）：

```json
{
  "dependencies": {
    "express": "^5.1.0"
  }
}
```

`dependencies` 就是"这个项目依赖（用到）了哪些包"的清单，`express` 是包名，`^5.1.0` 是版本号。同时目录里多出一个 `node_modules` 文件夹，express 的代码就下载在这里面——你的代码里 `require('express')` 或者 `import express from 'express'`，实际读的就是这个文件夹里的东西。

装的时候不加包名，就是照着 `package.json` 里现有的清单，把所有包都装一遍：

```bash
npm install
#   不带包名：批量安装 package.json 里记录的所有依赖
#   适用场景：把别人的项目 clone 下来之后，第一步就是跑这个
```

## pnpm 是什么：另一个包管理器，用法基本对得上

pnpm 也是一个包管理器，干的事情和 npm 一模一样——读清单、下载包、让你的代码能用上它们。区别在实现方式：pnpm 把下载过的包统一存了一份，项目里用的是指向这份存货的链接，而不是每个项目都单独复制一份。这样做的具体好处，下一节结合实际例子讲。

pnpm 不随 Node.js 自带，需要单独装一次。官方目前推荐这两种方式（[pnpm.io/installation](https://pnpm.io/installation)）：

```bash
# 方式一：官方安装脚本，跟 Node.js 版本无关，装完可以直接用
curl -fsSL https://get.pnpm.io/install.sh | sh -

# 方式二：已经有 npm 的话，用这条更省事（要求 Node.js 22.13 及以上）
npx get-pnpm
#     npx：临时下载并运行一个包，不会把它留在项目里
```

装完用 `pnpm -v` 确认一下能不能用。接下来重复一遍刚才 npm 的例子，换成 pnpm：

```bash
mkdir pnpm-demo && cd pnpm-demo

pnpm init
#   跟 npm init -y 效果一样，生成默认的 package.json，pnpm 这条命令不用额外加 -y

pnpm add express
#   add：pnpm 里"装一个包"用的动词是 add，不是 install
#   效果跟 npm install express 一样：下载、记进 package.json
```

`package.json` 里同样会多出 `dependencies` 字段，跟 npm 那份长得一样。批量安装现有清单用：

```bash
pnpm install
#   跟 npm install 不带包名一样：装 package.json 里记录的所有依赖
```

到这里为止，npm 和 pnpm 做的事情、结果对项目本身的影响是一样的：项目能正常 `require`/`import` 到 express。差别要打开 `node_modules` 才看得出来，下面细讲。

## 常用命令对照

日常会用到的命令，两边基本是一对一的关系：

| 要做的事 | npm | pnpm |
| --- | --- | --- |
| 初始化一个新项目 | `npm init -y` | `pnpm init` |
| 装 package.json 里的所有依赖 | `npm install` | `pnpm install`（可简写 `pnpm i`） |
| 装一个新的包 | `npm install express` | `pnpm add express` |
| 装一个只在开发时用的包 | `npm install -D vitest` | `pnpm add -D vitest` |
| 卸载一个包 | `npm uninstall express` | `pnpm remove express`（可简写 `pnpm rm`） |
| 跑 package.json 里定义的脚本 | `npm run build` | `pnpm build`（可以省掉 `run`） |
| 临时用一个包，不装进项目 | `npx create-vite` | `pnpm dlx create-vite` |
| 全局装一个命令行工具 | `npm install -g typescript` | `pnpm add -g typescript` |

`-D` 是"这个包只在开发时需要，上线运行不需要"的意思，比如测试工具、代码格式化工具，这类包会被记进 `package.json` 的 `devDependencies` 里，跟 `dependencies` 分开。

## 装出来的东西为什么长得不一样

前面装的都是同一个包（`express@5.1.0`），用 npm 和 pnpm 分别装一遍，打开 `node_modules` 看到的结构完全不同。

npm 装完之后，`node_modules` 顶层是这样：

```bash
$ ls node_modules
accepts/          body-parser/     bytes/       call-bind-apply-helpers/
call-bound/       content-disposition/          content-type/
cookie/           cookie-signature/             debug/       depd/
...               express/         ...          （共 65 个文件夹）
```

只装了一个 express，`node_modules` 顶层却出现了 65 个文件夹。原因是 express 自己也用到了别的包（比如 `body-parser`、`cookie`），那些包又用到了更多别的包，npm 把这些"包的包"全都摊平放在了同一层，图的是目录结构简单、不用一层套一层地嵌套。

pnpm 装完同样的项目，`node_modules` 顶层是这样：

```bash
$ ls -la node_modules
drwxr-xr-x  .pnpm/
lrwxr-xr-x  express -> .pnpm/express@5.1.0/node_modules/express
```

顶层只有一个 `express`（而且它是个快捷方式，指向 `.pnpm` 文件夹里的真实位置），你在 `package.json` 里写了什么包，顶层就只显示什么包。express 依赖的那些包，都被挪进了 `.pnpm` 这个单独的文件夹里，不摆在最外层。

这个差别看着只是"整不整洁"，但会带来两个实际后果。

### 后果一：npm 项目里能用到你没装过的包

Node.js 找一个包的规则很简单：从当前文件往上，找最近的一层 `node_modules`，看里面有没有这个名字的文件夹。npm 把所有包都摊平放在顶层，这就导致一个问题：即使你的项目**没有**声明过某个包，只要它恰好是 express 之类的包间接带进来的，你的代码照样能用。

实测验证一下。项目的 `package.json` 里只声明了 `express`：

```bash
$ node -e "require('debug'); console.log('用上了')"
用上了
```

`debug` 这个包从没在 `package.json` 里出现过，但因为 express 依赖它、npm 又把它摊平放在了顶层，`require('debug')` 照样能成功。这种"代码里用了、但项目清单里没声明"的包，业内管它叫**幽灵依赖**。

它为什么算个问题：你能用上 `debug`，完全是因为 express 恰好也在用它。哪天 express 升级，换了别的日志库，或者干脆不用 `debug` 了，你那行 `require('debug')` 会突然报错——而你自己的 `package.json` 一行没改。出问题的时候会很懵：明明我什么都没动。

同样的代码，在 pnpm 装出来的项目里：

```bash
$ node -e "require('debug')"
Error: Cannot find module 'debug'
```

直接报错，因为顶层的 `node_modules` 里压根没有 `debug` 这个文件夹（它被挪进了 `.pnpm` 深处，Node.js 按规则找不到）。这个报错是好事——它逼着你老老实实把用到的包都写进 `package.json`，而不是稀里糊涂地蹭到别人的依赖。

### 后果二：npm 项目里，同一个包可能被存好几份

再往深一层看：express 依赖的一堆包里，恰好有两个包各自依赖了不同版本的 `content-type` 这个包（1.0.5 和 2.1.0）。npm 只能把其中一个版本摊平放顶层，另一个版本就地留在依赖它的那个包自己的文件夹里，实测结果是这样：

```
1.0.5  node_modules/content-type/                          ← 摊平到顶层的版本
2.1.0  node_modules/body-parser/node_modules/content-type/
2.1.0  node_modules/type-is/node_modules/content-type/
2.1.0  node_modules/negotiator/node_modules/content-type/
```

`content-type@2.1.0` 这一个版本，在磁盘上被复制了三份。同一份代码存了三份，不只是多占磁盘——如果这个包内部会记一些状态（比如缓存），这三份是三个互不相通的独立实例，调试起来容易摸不着头脑。

pnpm 装出来的同一个项目，`content-type` 只有两个文件夹，一个版本各占一个：

```
node_modules/.pnpm/content-type@1.0.5/
node_modules/.pnpm/content-type@2.1.0/
```

没有多余的复制。原因还是那个 `.pnpm` 文件夹：pnpm 把每个"包名+版本"的组合只存一份，谁要用哪个版本，就从这唯一的一份那里拉一条快捷方式过去，而不是复制整个包。

## pnpm 顺带更省磁盘、装得更快吗

理论上会更省磁盘——同一份包只存一份，而不是像 npm 那样可能存好几份重复的。pnpm 能做到这一点，靠的是一个全局的包仓库，具体路径因系统而异，可以用一条命令查到：

```bash
$ pnpm store path
/Users/xxx/Library/pnpm/store/v10
```

不管哪个项目要用某个包，都是从这个仓库里"借用"，不用重新下载、也不用真的复制一份文件。

不过这只是原理，实际数字要拿出来看才算数。同一个只装了 `express` 的项目，实测对比：

| 指标 | npm 11.4.0 | pnpm 10.15.1 |
| --- | --- | --- |
| `node_modules` 占用磁盘 | 3.9M | 3.8M |
| 第一次安装耗时 | 2.29 秒 | 8.90 秒 |
| 再装一个同样依赖的新项目 | 0.88 秒 | 1.73 秒 |

如实说：**在这种只有几十个包的小项目上，pnpm 并不比 npm 快**，反而更慢一点——因为它要维护那个全局仓库、计算文件的校验值，这些工作在项目很小的时候占的比例更高。磁盘占用上两者也没差多少。

pnpm 真正体现优势的地方是大项目：依赖包多到几百上千个的时候，"同一份包不用重复复制"能省下的磁盘和时间才会明显。项目小的话，选它主要是图上一节讲的那个好处——不会稀里糊涂用上没声明过的包。

## 新手该用哪个

刚开始学、或者项目很小，直接用 npm 就够了——它跟着 Node.js 一起装好了，不用操心额外安装，网上找到的教程、别人的项目也大多默认你用 npm，兼容性最没有争议。

等到项目变大、依赖变多，或者开始在意"项目声明的依赖和实际用到的依赖对不上"这类问题时，可以考虑换成 pnpm。好在两者的命令基本一一对应（参考上面那张表），迁移成本很低——把 `npm install` 换成 `pnpm install`，`npm install <包名>` 换成 `pnpm add <包名>`，大部分项目就能正常跑起来了。
