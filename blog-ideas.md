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
