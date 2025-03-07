---
layout: post
title: 3分钟搭建GPT3-QQ机器人！
date: 2022-12-18 00:49:20
categories:
    - other
---

# 3.3日更新: 支持gpt-3.5-turbo模型

本文部分已经过时，关于**插件配置**部分请直接看仓库readme配置

https://github.com/chrisyy2003/nonebot-plugin-gpt3#%E9%85%8D%E7%BD%AE

# 遇到问题？

1.  私聊有反应，群里没反应？

可能是机器人被风控，二是群中触发根据设置可能需要@

3.   linux安装不上

插件依赖playwright，**playwright在linux上只支持ubuntu系统，不支持centos**。

4.   开启图片渲染失败

请执行命令，`pip install playwright && playwright install`，并在 `.env.dev` 文件中设置 `FASTAPI_RELOAD=false`

# 前言

## chatGPT和GPT3的区别

很多小伙伴可能对二者概念有些模糊，其实这是两个不同的东西。

ChatGPT 和 GPT-3 都是由 OpenAI 训练的大型语言模型，但它们有一些关键的区别。GPT-3，即**Generative Pretrained Transformer 3**，是 OpenAI 的第三代 GPT 语言模型，是目前最强大的语言模型之一。它可以针对各种自然语言处理任务进行微调，包括语言翻译、文本摘要和问答。

另一方面，ChatGPT 是**专门为聊天机器人应用程序设计的 GPT-3 模型的变体**。它已经在大型对话文本数据集上进行了训练，因此能够生成更适合在聊天机器人上下文中使用的响应。ChatGPT 还能够在对话中插入适当的特定于上下文的响应，使其更有效地保持连贯的对话。

在性能方面，**ChatGPT 不如 GPT-3 强大**，但它更适合聊天机器人应用，这使其成为在实时聊天机器人系统中使用的更好选择。总体而言，ChatGPT 和 GPT-3 都是强大的语言模型，但它们的设计目的不同，各有优缺点。

GPT3是公开了官方API接口，可以通过openai官方仓库找到代码实现。

而ChatGPT目前是没有公开API，所有ChatGPT的机器人都是通过浏览器抓包，逆向解析API接口实现的，这也导致了openAI上了CloudFlare导致的严重antiBot限制。

## openAI限制时间线

2022 年 12 月 12 日：OpenAI 为其 API 添加了 Cloudflare 保护，添加了cf字段，限制登陆ip和请求ip相同

2022 年 12 月 13 日：一个临时方案是通过手动复制cf，token值请求，cf字段时限两小时。

2022 年 12 月 14 日：添加了动态cf字段，请求会使得cf字段刷新。

2022 年 12 月 15 日：限制了ip，非可用地区只能请求一句话，方案变为了完全的模拟浏览器的方案。

## 安装

启动bot有两种方式：

1.  docker安装（推荐）
2.  手动安装

手动安装总共只有两步：

1.  下载整合包，输入qq号登陆
2.  安装插件，输入API

前两步实际上包含cqhttp和nonebot的配置，但是我整合了一下，可以使得大家更方便一点，可以通过这里直接下载。

**并且下文则按照整合包的安装顺序讲解。**

如果您了解相关原理，可以不通过打包的文件，这样可以实现更加自定义的配置，具体可以看前一个文章，其中有讲到如何配置cqhttp和nonebot

# docker安装（推荐）

首先[下载整合包](https://file.chrisyy.top/qq-example.zip)，随后解压，切换到解压文件夹根目录，

1.  在`config/chatgpt_api_key.yml`填入自己的API，
2.  随后在`bot.py`中加上如下代码，加载插件

```
nonebot.load_plugin('nonebot_plugin_gpt3')
```


最后执行以下命令即可

```
docker run -v $(PWD):/bot -d -p 8080:8080 chrisyy2003/nonebot bash /bot/start.sh
```

随后在`cqhttp/config.yml`中修改为您的QQ号，随后根据系统启动相应的cqhttp程序即可。cqhttp启动方式可以参看下面的步骤。

>   cqhttp请不要在docker内启动，因为docker无法保存数据，不然每次需要重新登陆。
>
>   docker只是提供一个nonebot的运行环境。docker内的文件的是双向绑定的，外面修改了文件，docker也会被修改，所以修改文件最好都是在当前目录中修改。

# 手动安装

## 启动框架

这一节会启动我门的QQ和一个最基本的机器人

1.  首先[下载整合包](https://file.chrisyy.top/qq-example.zip)，随后解压，解压出来的文件应该如下：

```
.
|-- .env
|-- .env.dev
|-- bot.py
|-- cqhttp
|-- pyproject.toml
`-- src
```

其中包含cqhttp文件夹和bot.py文件和一些配置文件。

随后在`cqhttp/config.yml`中修改为您的QQ号，随后根据系统启动相应的cqhttp程序即可。

windows推荐在powershell下运行，启动方式，在命令行`./go-cqhttp-[你的系统]`然后enter即可。

![image-20221218173930474](./qq-gpt3/image-20221218173930474.png)

2.   第二步安装相关的依赖库。

```
pip install nonebot2
pip install nonebot-adapter-onebot
```

3.   启动bot

在文件夹的根目录输入一下命令即可启动bot

```
python3 bot.py
```

如果启动成功，可以看到如下的日志

![image-20221217114837304](./qq-gpt3/image-20221217114837304.png)

这里有几个部分说明：

1.  第一个框，dev表示我们进入的是dev环境，读取的配置是env.dev，如果是prod请看env文件是否正确设置了环境为dev。
2.  第二个框，表示读取了config了的配置，如果发现自己的指令没有触发，请检查这里是否正确读取。
3.  第三个框，表示我们成功加载了那几个插件，**这里我们加载了内置的echo插件**。
4.  第四个框，表示我们的nonebot和cqhttp成功链接，cqhttp的消息能够转发到nonebot啦！

随后要与机器人互动，可以首先使用`echo`命令让其输出一些信息。

![image-20221219122546182](./qq-gpt3/image-20221219122546182.png)

## 插件安装

通过包管理器安装，可以通过pip，或者poetry等方式都安装，我们这里以pip为例，在命令行中输入：

```
pip install nonebot-plugin-gpt3 playwright -U && playwright install
```

随后在`bot.py`中加上如下代码，加载插件

```
nonebot.load_plugin('nonebot_plugin_gpt3')
```

随后我们可以启动bot，通过`python3 bot.py`来启动，首次启动会在默认路径API路径`config/chatgpt_api_key.yml`下生成yml文件，在其中可以配置您的API，如果已经存在了则会直接读取，所以首次启动会加载0个API。

![image-20221218172400814](./qq-gpt3/image-20221218172400814.png)

需要填入API后再次启动就可以发现成功加载了API

![](./qq-gpt3/image-20221218172520895.png)

![image-20221218172611773](./qq-gpt3/image-20221218172611773.png)

## windows

windows的用户还额外需要安装一个rust的环境，[点击这里下载](https://file.chrisyy.top/rustup-setup.exe)之后，安装即可。

插件的更多配置，请看插件主页：

https://github.com/chrisyy2003/nonebot-plugin-gpt3

# 使用

## 基本会话

对话前，加上前缀即可与GPT3对话。

​	![image-20221221203808576](./qq-gpt3/image-20221221203808576.png)

## 连续会话

输入**chat/聊天/开始聊天**即可不加前缀，连续的对话，输入**结束/结束聊天**，即可结束聊天

![img](./qq-gpt3/68747470733a2f2f636872697379792d696d616765732e6f73732d636e2d6368656e6764752e616c6979756e63732e636f6d2f696d672f696d6167652d32303232313231373233303035383937392e706e67.png)

## 人格设置

预设了**AI助手/猫娘/nsfw猫娘**三种人格，可以通过人格设置切换。内置的设定可以从[这里看到](https://github.com/chrisyy2003/nonebot-plugin-gpt3/blob/main/nonebot_plugin_gpt3/__init__.py#L9-L11)。

![img](./qq-gpt3/68747470733a2f2f636872697379792d696d616765732e6f73732d636e2d6368656e6764752e616c6979756e63732e636f6d2f696d672f696d6167652d32303232313231373233313730333631342e706e67.png)

同样也可以手动指定人格

![img](./qq-gpt3/68747470733a2f2f636872697379792d696d616765732e6f73732d636e2d6368656e6764752e616c6979756e63732e636f6d2f696d672f3230323330333036313533323632362e706e67.png)

## 图片渲染

图片渲染可以在配置文件中配置是否，需要渲染

图片渲染需要安装额外的依赖，具体看插件主页：

![img](./qq-gpt3/68747470733a2f2f636872697379792d696d616765732e6f73732d636e2d6368656e6764752e616c6979756e63732e636f6d2f696d672f696d6167652d32303232313231373233333732393236332e706e67.png)
