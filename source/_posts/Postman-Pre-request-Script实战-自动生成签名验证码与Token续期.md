---
title: "Postman Pre-request Script 实战：自动生成签名、验证码与 Token 续期"
date: 2026-07-28 22:00:00
cover: /img/p64.jpg
categories: tools
tags:
  - Postman
  - API测试
  - 效率工具
  - JavaScript
description: "MD5 签名、Google Authenticator 验证码、Token 续期，三个 Pre-request Script 案例同一类坑：不报错，值不对。"
---

联调一个第三方机器人对接接口，对方要求每个请求带 MD5 签名。手动拼字符串算签名这种事显然应该丢给 Pre-request Script 自动做——写完之后觉得这套思路挺好用，接下来测后台管理接口的时候，登录要输 Google Authenticator 动态验证码、鉴权 token 会过期，干脆也一并自动化掉。

三个场景表面上不一样，用的却是同一个套路：**脚本里现算一个值，存成变量，请求发出去的时候引用这个变量。** 也正因为套路一样，踩的坑也是同一类——脚本本身从头到尾没有报一次错，跑得很顺利，产出的值就是不对，报错信息还完全是另一件事（"参数错误""密码错误"），排查起来比直接崩溃的 bug 麻烦得多。这篇假设你已经知道 Pre-request/Post-response 脚本和 `pm` 对象是什么——不熟悉的话先看[Postman 进阶技巧这篇](/Postman进阶技巧-自动化认证与批量测试实战)里的脚本语法速览。

<!-- more -->

## 案例一：签名自动计算，栽在动态变量身上

签名接口的规则很典型：取所有非空参数，按字典序拼成 `k1=v1&k2=v2` 的形式，末尾加上密钥，MD5 之后转大写。Pre-request Script 里写：

```javascript
const key = pm.environment.get("api_key");
const resolvedBody = pm.variables.replaceIn(pm.request.body.raw);
const params = JSON.parse(resolvedBody);

const keys = Object.keys(params)
    .filter(k => k !== "signature" && params[k] !== "")
    .sort();

const signStr = keys.map(k => `${k}=${params[k]}`).join("&") + `&key=${key}`;
pm.environment.set("signature", CryptoJS.MD5(signStr).toString().toUpperCase());
```

Body 里对应字段留 `{{signature}}` 占位，为了让每次测试用的账号、流水号都不重复，我图省事直接在 body 模板里塞了 Postman 的动态变量：

```json
{
    "account": "{{$randomUserName}}",
    "request_no": "{{$randomUUID}}",
    "signature": "{{signature}}"
}
```

发出去之后，服务端稳定返回"参数错误"（对方约定的通用错误码，签名校验失败也会走这个码，不会单独告诉你是签名的问题）。账号密码对不上号可以理解，签名怎么会一直错？

关键在 `{{$randomUserName}}`、`{{$randomUUID}}` 这类**动态变量每次被解析都会重新生成一个新值**，不是普通变量那种存进去就固定不变的东西。脚本里 `pm.variables.replaceIn(pm.request.body.raw)` 解析了一次 body（顺带把动态变量算成随机值 A），拿这份 A 去算的签名；但 Postman 真正把 body 模板渲染成最终请求体发出去的时候，**又重新解析了一次**，这次算出来的是随机值 B。签名算的是 A，实际发出去的是 B，服务端一校验必然不通过——而且这个不一致完全不会体现在任何日志里，两次解析都在 Postman 内部悄悄完成，从请求日志上看 account/request_no 的值是"对的"（能正常显示出来），只是跟签名时用的不是同一份。

解法是不在 body 里直接引用动态变量，先在脚本里手动解析一次、存成普通环境变量，body 引用这个普通变量：

```javascript
pm.environment.set("account", pm.variables.replaceIn("{{$randomUserName}}"));
pm.environment.set("request_no", pm.variables.replaceIn("{{$randomUUID}}"));
```

普通环境变量不会重复随机化，脚本算的和最终发送的永远是同一个值。

## 案例二：Google Authenticator 验证码自动生成，栽在一行看起来没问题的代码上

后台管理接口登录要求账号、密码，外加一个 Google Authenticator 的 6 位动态验证码。手动开手机 App 抄一遍太烦，其实按 [RFC 6238](https://datatracker.ietf.org/doc/html/rfc6238) 的算法自己算就行——HMAC-SHA1、30 秒一个周期、取哈希最后一个字节的低 4 位当偏移量做动态截断：

```javascript
function base32Decode(base32) {
    // 解出一个 0~255 的字节数组，Base32 解码本身没有问题
    const alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567";
    base32 = base32.replace(/=+$/, "").toUpperCase();
    let bits = "";
    for (const c of base32) {
        const val = alphabet.indexOf(c);
        if (val !== -1) bits += val.toString(2).padStart(5, "0");
    }
    const bytes = [];
    for (let i = 0; i + 8 <= bits.length; i += 8) {
        bytes.push(parseInt(bits.substr(i, 8), 2));
    }
    return bytes;
}

function generateTOTP(secretBase32, period = 30, digits = 6) {
    const key = CryptoJS.lib.WordArray.create(base32Decode(secretBase32)); // 第一版写法，问题出在这一行
    const counter = Math.floor(Date.now() / 1000 / period);
    // ...省略计数器编码和 HMAC 部分
}
```

第一次测登录，报的是"密码错误"。查下来发现是我自己搞错了状态——数据库里那行账号的密码哈希跟我以为的明文对不上，重置一下密码就过去了，跟脚本没关系。

密码这关过了之后，稳定卡在"验证码错误"。这就有意思了：账号密码都验证过是对的，验证码逻辑上应该没道理一直错。第一反应是怀疑时钟——TOTP 依赖系统时间，客户端和服务端时钟差太多确实会导致验证码校验失败。但用 `date -u` 分别在跑 Postman 的机器和跑服务的机器上核对了一遍，两边时间完全一致，误差在一秒以内，排除。

真正定位到问题是靠一次"绕开 Postman、直接验证"：拿服务端语言自己的 TOTP 库，按同一个密钥、当前这一刻的时间戳单独算一次正确验证码，再拿这个验证码直接 `curl` 打服务端接口——**登录成功了**，说明账号、密码、密钥、服务端逻辑全都没问题，问题只可能出在 Postman 脚本这一层算出来的验证码本身就是错的。

再一步定位：把脚本用到的 `secret`、算出来的 `code`、以及 `Date.now()` 对应的时间戳都打到 Postman Console 里，拿这个精确时间戳去服务端库重新算一次"标准答案"，跟 Postman 输出的验证码对比——完全对不上，不是差个位数或者边界抖动那种"接近但不对"，是两个毫不相干的六位数。同样的密钥、同样的时间戳，Postman 算出来的和标准库算出来的应该逐字节一致才对。

问题出在这一行：

```javascript
const key = CryptoJS.lib.WordArray.create(byteArray); // byteArray 是一个 0~255 的普通数组
```

看起来很合理——把字节数组喂给 `WordArray.create`，应该就构造出了对应的 WordArray。但 `CryptoJS.lib.WordArray.create()` 接收的参数被当成一个 **32 位 word 的数组**，不是字节数组：数组里每一项都会被当成一个完整的 4 字节 word 使用，而不是被打包进某个 word 里的一个字节。传一个字节数组进去，构造出来的 WordArray 内容整个错位，而且这个错误**不会抛任何异常**，脚本正常跑完、正常返回一个六位数，从表现上完全看不出问题——直到拿这个数字去服务端验证，一直被拒才发现。

为了确认真的是这里的问题，把 Postman 内置的同一个 `crypto-js` 库单独装到本地（`npm install crypto-js`），拿同一个密钥、同一个时间戳，跑一遍这行错误写法的完整逻辑——算出来的六位数跟 Postman Console 打印出来的**一模一样**，坐实了就是这一行。这个交叉验证的思路本身值得记一下：怀疑某个库用法有问题、又没法直接在报错信息里看出来的时候，把同一个库单独装到 Node 环境里跑一遍相同逻辑，比对着文档逐字读代码快得多。

正确写法是不要直接把字节数组塞给 `WordArray.create`，转成十六进制字符串，用 `CryptoJS.enc.Hex.parse(hexString)` 构造——这是 CryptoJS 官方文档里"从原始字节构造 WordArray"推荐的方式，两个字符对应一字节，不存在对齐歧义。`base32Decode` 已经能拿到字节数组，只需要再加一个小工具函数把它转成十六进制字符串：

```javascript
function bytesToHex(bytes) {
    return bytes.map(b => b.toString(16).padStart(2, "0")).join("");
}

function generateTOTP(secretBase32, period = 30, digits = 6) {
    const key = CryptoJS.enc.Hex.parse(bytesToHex(base32Decode(secretBase32)));

    const counter = Math.floor(Date.now() / 1000 / period);
    const counterWordArray = CryptoJS.enc.Hex.parse(counter.toString(16).padStart(16, "0"));

    const hmac = CryptoJS.HmacSHA1(counterWordArray, key);
    const bytes = CryptoJS.enc.Hex.stringify(hmac).match(/.{2}/g).map(h => parseInt(h, 16));

    const offset = bytes[bytes.length - 1] & 0xf;
    const bin = ((bytes[offset] & 0x7f) << 24) |
                ((bytes[offset + 1] & 0xff) << 16) |
                ((bytes[offset + 2] & 0xff) << 8) |
                (bytes[offset + 3] & 0xff);

    return (bin % Math.pow(10, digits)).toString().padStart(digits, "0");
}

pm.environment.set("google_code", generateTOTP(pm.environment.get("google_secret")));
```

换成这版之后，跟标准库对比逐字节一致，登录一次通过。

## 案例三：Token 过期自动续期，两个小坑

前面两个案例分别解决了"这个请求要带一个现算的值"，token 续期是同一个套路往前再走一步：整个 Collection 共用一份 Pre-request Script，请求发出去之前先检查 token 是不是快过期了，是的话自动跑一遍登录（复用案例二那套 TOTP 生成逻辑）：

```javascript
(async () => {
    if (pm.request.url.toString().includes("/login")) return; // 登录请求自己跳过，避免套娃

    function isTokenValid(token) {
        if (!token) return false;
        try {
            const payload = JSON.parse(atob(token.split(".")[1].replace(/-/g, "+").replace(/_/g, "/")));
            return payload.exp && payload.exp * 1000 > Date.now() + 60000; // 留 60 秒余量
        } catch (e) {
            return false;
        }
    }

    if (!isTokenValid(pm.environment.get("admin_token"))) {
        const code = generateTOTP(pm.environment.get("google_secret")); // 复用案例二的函数

        const res = await pm.sendRequest({
            url: pm.environment.get("base_url") + "/login",
            method: "POST",
            header: { "Content-Type": "application/json" },
            body: {
                mode: "raw",
                raw: JSON.stringify({
                    account: pm.environment.get("admin_account"),
                    password: pm.environment.get("admin_password"),
                    google_code: code
                })
            }
        });

        const data = res.json();
        if (data.success) {
            pm.environment.set("admin_token", data.data.token);
        } else {
            console.error("自动登录失败:", data);
        }
    }

    pm.request.headers.upsert({ key: "Authorization", value: "Bearer " + pm.environment.get("admin_token") });
})();
```

第一个坑是 `await pm.sendRequest(...)` 直接写在脚本最外层：`pm.sendRequest` 配合顶层 `await` 是比较新的 Postman 版本才支持的写法，老版本会当场语法报错——而且一报错**整个 Pre-request Script 都不会往下执行**，连脚本最后加 `Authorization` 头那一行都跑不到，表现就是这个请求直接没带 token 发出去。保险起见统一包一层 `(async () => { ... })();`，新老版本都兼容。

第二个坑更隐蔽：改完之后自动登录还是失败，Console 里打出来的失败响应显示的是"认证失败"这类跟密码、验证码都不沾边的错误。查了半天，最后发现是 `pm.environment.get("admin_account")` 这个变量名跟 Environment 面板里实际存的名字对不上——脚本里写的是下划线 `admin_account`，面板里当初存的是中划线 `admin-account`。

`pm.environment.get` 读一个不存在的变量名**不会报错**，只会返回 `undefined`；而 `JSON.stringify({ account: undefined, password: "..." })` 会**静默地把值是 `undefined` 的字段整个丢掉**，不会变成 `null`，也不会有任何提示。所以脚本实际发出去的请求体里根本没有 `account` 这个字段，服务端收到一个空账号，返回的错误信息自然跟"密码"或"验证码"都对不上——这类报错最容易把人带偏去查错方向，因为报错文案暗示的问题根源和真实原因完全是两码事。

养成的习惯是：写脚本引用 `pm.environment.get(...)` 之前，先去 Environment 面板核对一遍变量名的精确拼写——大小写、下划线还是中划线、有没有多余空格，这一步核对的成本，远低于事后对着一个文不对题的错误信息排查半天。

## 脚本不报错时，靠这三条路子排查

三个场景（签名、验证码、token）用的是同一套脚本模式，踩的坑也是同一类问题：**脚本本身不报错，产出的值却是错的，或者字段根本没传到请求里**。这类问题排查起来比直接崩溃的 bug 更费时间，因为常规的"看报错信息"完全帮不上忙——报错信息要么是通用错误码，要么指向一个跟真实原因毫不相关的方向。

真正管用的排查方式是三条：把每一步的中间值（原始密钥、拼出来的待签名字符串、算出来的哈希、精确的时间戳）用 `console.log` 一层层打出来；怀疑某个库用法不对时，拿同一个库单独装到本地跑一遍相同逻辑做交叉验证，而不是对着文档猜；涉及跨端一致性的问题（这里是签名、这里是密钥、这里是变量名），把两边实际用的值一个字符一个字符地摆在一起对比，而不是假设"看起来应该是对的"。
