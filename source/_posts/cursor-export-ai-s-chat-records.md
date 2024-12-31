---
title: 'cursor: 导出聊天会话记录'
date: 2024-12-31 17:21:26
categories: cursor
tags:
  - chat聊天记录导出
---


# 背景

自从ai出现以来，极大的提高了我的生产效率，特别是内置集成ai的`IDE`出现之后。虽然对于编码方面的提效非常
明显，但是对于**IDE功能使用体验**上来说，有些地方个人还是觉得需要改进。
比如，目前最火的`cursor`，内置ai，特色功能就是基于代码库的chat对话，以及行内代码提问。
然后，这个chat功能使用多次之后，想要**记录下来这些信息**，会发现很麻烦，因为目前官方**不支持chat记录的导出**。

在一番搜索之后，找到一些不是办法的办法。

<!-- more -->

# 解决方案

大致思路是：
1. 写一个脚本读取`cursor`的`chat`本地的数据文件。
2. 然后指定格式导出。

我使用的这个py脚本：[Cursor Chat Export](https://github.com/somogyijanos/cursor-chat-export)，相关
操作在readme中都有说明。
注意：
- 这个需要安装python环境；相关依赖下载`pip install -r requirements.txt`即可。
- 这个脚本读取的cursor数据路径配置在`config.yaml`中，可以根据自己的需求进行修改（如果指定了路径，可以忽略）。
- 脚本读取的是最近的数据，所以如果想要指定某个chat的记录，需要使用到**搜索功能**，但是这个功能目前使用起来有点鸡肋：
  - 因为按照内容关键字或者chat的title搜索，没有搜到想要的结果（有点懒，没有去研究改脚本）。
  - 在`Windows`运行，命令是：` D:\python\python.exe ./chat.py`



希望官方还是尽快推出这个功能吧。

> 参考资料：
> [官方论坛相关帖子讨论1](https://forum.cursor.com/t/chat-history-folder/7653/2)
> [官方论坛相关帖子讨论2](https://forum.cursor.com/t/how-do-i-export-chat-with-ai/144)
> [相关工具](https://github.com/somogyijanos/cursor-chat-export)

