# Blog Ideas

在任何 Claude Code session 里发现值得写的点，直接追加到这里。

格式随意，简单记录即可：

```
## YYYY-MM-DD：主题关键词

- 想写的点 1
- 想写的点 2
- 参考：某个 session / 某段对话 / 某个链接
```

---

<!-- 从这里开始记录 -->

## 2026-07-02：Docker 实用教程

**背景**

Docker 入门教程到处有，但大多停留在"跑个 hello-world"。写一篇面向后端开发者的实用教程，聚焦真实开发场景：本地起服务、打镜像、多容器联调。

**可以覆盖的场景**

- 核心概念：镜像 vs 容器、层（layer）缓存、registry
- 常用命令速查：`run`、`exec`、`ps`、`logs`、`stop`、`rm`、`build`、`pull`/`push`
- Dockerfile 写法：FROM/RUN/COPY/ENV/EXPOSE/CMD 各指令含义，多阶段构建减小镜像体积
- docker-compose：本地起 MySQL + Redis + 应用服务，volumes 持久化数据，网络互通
- 实际场景：Go 应用打镜像（含 .dockerignore）、挂载本地目录热更新、查看容器内日志
- 常见问题：容器里访问宿主机服务（`host.docker.internal`）、端口映射、镜像清理

**博客可以展开的点**

- 用"本地开发环境容器化"为主线串联所有命令，而不是逐命令罗列
- docker-compose.yml 给一个完整可跑的例子（Go + MySQL + Redis）
- 说明什么时候用 Docker，什么时候不用（开发机直装 vs 容器化的取舍）

---

## 2026-07-02：Git 日常操作手册

**背景**

Git 命令多而散，很多人只记住 add/commit/push 三板斧，遇到合并冲突、撤销提交、整理历史就慌了。写一篇覆盖真实工作场景的 Git 操作参考，不是命令大全，而是按场景组织。

**可以覆盖的场景**

- 撤销操作：`git restore`、`git reset`、`git revert` 的区别和适用场合
- 暂存区操作：`git stash`（save/pop/list/apply）
- 分支管理：创建、切换、合并、删除、重命名
- 查看历史：`git log` 常用参数（`--oneline --graph --all`）、`git diff` 对比范围
- 合并 vs rebase：使用场景对比，rebase 的黄金法则（不对已推送的分支 rebase）
- 解决冲突：merge 冲突标记读法、工具辅助（git mergetool / lazygit）
- 远程操作：fetch vs pull、强推的风险与替代方案
- 实用小技巧：`git bisect` 二分查找问题提交、`git blame`、`git cherry-pick`

**博客可以展开的点**

- 每个场景配一个"踩过的坑"或"为什么这么做"的解释，不只列命令
- 用图或 ASCII 图示说明 merge vs rebase 的提交线变化
- 结合 lazygit / git-delta 这些工具提升体验

---

## 2026-07-01：Go 接口设计——当参数不够用时怎么办

**背景**

在接入第三方支付渠道时，遇到一个典型问题：现有的 `IPay` 接口定义是：

```go
type IPay interface {
    QueryOrder(tradeNo, total string) (map[string]interface{}, error)
}
```

新渠道的查单 API 需要同时传商户订单号和渠道订单号两个字段，但接口只有两个 string 参数，语义固定。

**权宜做法（能跑但有隐患）**

不改接口签名，在调用方对新渠道单独处理，把 `total` 参数借道传渠道订单号：

```go
// payment.go 调用方
if payChannel == "new_channel" {
    _, err = e.QueryOrder(params.OutTradeNo, params.ChannelTradeNo) // total 借用为渠道订单号
} else {
    _, err = e.QueryOrder(queryTradeNo, params.Money)
}

// new_channel.go 实现方
func (ep *NewChannelPay) QueryOrder(tradeNo, total string) (...) {
    params := map[string]interface{}{
        "order_sn":  tradeNo,
        "trade_num": total, // 按约定解读，实为渠道订单号
        ...
    }
}
```

能解决问题，但有隐患：参数语义靠约定维护，下次再来新渠道有额外参数，又得打同样的补丁。

**行业标准做法：Struct 参数**

把 `QueryOrder` 的参数改成结构体：

```go
type QueryOrderReq struct {
    OrderSN        string // 商户订单号
    ChannelTradeNo string // 渠道订单号
    Money          string // 金额
}

type IPay interface {
    QueryOrder(req QueryOrderReq) (map[string]interface{}, error)
}
```

各渠道实现只取自己需要的字段，调用方统一构造 req 一次传入。以后新增字段只需在 struct 里加一行，接口签名不变，已有渠道实现不受影响（Go struct 零值为空，老渠道拿到空字段不影响逻辑）。

**博客可以展开的点**

- 用支付渠道接入场景引入问题，读者有代入感
- 对比两种方案的改动面、可读性、扩展成本
- 说明 struct 方案为什么是"非破坏性变更"（零值兜底）
- 顺带提一下 Options 模式和 Context 透传作为其他行业选项
- 结论：接口参数一旦超过 2-3 个同类型值，就应该考虑 struct 化
