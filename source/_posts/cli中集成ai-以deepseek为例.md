---
title: cli中集成ai,以deepseek为例
date: 2025-01-15 14:55:47
categories: 
  - ai
  - deepseek
tags:
  - api
  - tokens
  - cli tool
---

最近国内的DeepSeek爆发出强劲的势头，纵观网友们的评价下来，很少有国产AI能有这样不错的评价。再加上最近DeepSeek注册就送500百万tokens，还有1块钱一百万tokens的这样的优惠力度，不得不去体验了一下。
[deepseek官网](https://www.deepseek.com/)

<!-- more -->

![官网公布的基准测试](../images/2025-01-15-15-17-27.png)
![模型&价格](../images/2025-01-15-15-24-16.png)
![赠送的tokens](../images/2025-01-15-15-25-09.png)


# 概述

目前来讲，我并没有很强的tokens使用场景，大多数以网页的chat聊天为主。
基于这种场景下的有个**痛点**就是：在终端和网页之间来回切换，有点浪费时间，不太友好。
所以我想着能不能把ai集成到cli中，让我在终端中可以直接和ai聊天，不需要切换到网页。

经过搜索，方案是可行的，工具采用的是[LLM](https://github.com/simonw/llm).

> 在一切配置开始前，需要如下前置条件：
> 1. 安装python环境
> 2. 注册deepseek账号

上面两个是最基本的条件，如果不满足建议谷歌搜索一下，这里不在赘述。

# 配置

## 安装LLM

用`pip`安装:
```bash
pip install llm
```

以后有需要更新版本，可以使用：
```bash
pip install -U llm
```

## 获取DeepSeek的API Key

1. 前往[deepseek官网](https://www.deepseek.com/)，注册一个新的账户。
2. 在[API Keys](https://platform.deepseek.com/api_keys)页面，点击**创建 API key**按钮，复制API Key，这是你的私钥，最好不要外泄。（注意：窗口一旦关闭后，API Key就会看不见，一定要保存好！当然如果没了，直接重新创建一个也可以）

## 配置LLM

由于LLM是基于OpenAI的，所以需要设置模型采用DeepSeek的模型。

1. 配置要使用模型的名称，这里是`deepseek`，名字随便取一个即可：
```bash
llm keys set deepseek
```
回车之后，会提示你输入你的API Key，输入你的API Key回车即可。不确定可以采用`dirname "$(llm logs path)"`找到`keys.json`文件，查看里面的API Key是否正常。

1. 配置默认使用的模型名称：
```bash
# 这里deepseek是你第一步输入的名称
llm models default deepseek
```
1. 配置请求`deepseek api`：
     * `dirname "$(llm logs path)"`，会输出llm的配置路径
     * 在这个路径下新建一个`extra-openai-models.yaml`文件，添加如下内容:
       ```yaml
        - model_id: deepseek
          model_name: deepseek-chat
          api_base: "https://api.deepseek.com/v1"
          api_key_name: deepseek
       ```
      * 如果你第一步的模型名称不是deepseek，这里的`model_id`和`api_key_name`需要改成第一步设置的名称。
      * `model_id`、`api_key_name`都是LLM里面概念，而`model_name`才是请求对应厂商的模型名称
2. 如果第二步没有设置成默认模型，也可以采用参数选用模型：
```bash
llm -m your_model_name "你的问题"
```


## 使用验证

终端下直接输入`llm "你的问题"`即可：
![验证api是否正常](../images/2025-01-15-16-06-59.png)
更多使用方法参考[官方文档](https://llm.datasette.io/en/stable/usage.html)





> 参考：
> https://github.com/simonw/llm
> https://llm.datasette.io/en/stable/setup.html
> https://llm.datasette.io/en/stable/other-models.html
> https://api-docs.deepseek.com/zh-cn/
> https://github.com/simonw/llm/issues/393