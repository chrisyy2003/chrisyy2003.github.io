---
layout: post
title: 5分钟在QQ群搭建ChatGPT机器人！
date: 2022-12-07 23:35:05
categories:
    - other
---

# openai上了Cloudflare临时解决方案

## 安装打包的bot

首先更新revchatGPT，`pip3 install revChatGPT --upgrade`

复制图中的cf_clearance的值
![image-20221212103154911](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212103154911.png)

随后配置修改为如下：

然后在https://github.com/chrisyy2003/lingyin-bot/blob/main/test.py#L8-L9修改为自己的token和cf cookie值和user agent值

User-agent的值看这个图

![image-20221212123411066](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212123411066.png)

随后运行[test.py](https://github.com/chrisyy2003/lingyin-bot/blob/main/test.py)文件（test文件同样也要修改，这是来检查你的信息能够正确的请求），如果test能跑过那么根据bot[主页](https://github.com/chrisyy2003/lingyin-bot)的说明安装bot就也能跑了

跑过指的是能返回如下的有效消息：

![image-20221212103356692](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212103356692.png)

test.py能运行就配置bot的env.dev就可以了，像这样

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212123451926.png)

# 存在的问题

可以直接看这里排查是否存在问题

1.  websocket bad handshake 
    请查看是否配置了v11适配器，在视频6：19秒处
2.  发送指令没有回复
    nonebot默认指令需要在前面加一个`/`斜杠来触发，请查看文章最后



# 前言

本文将带来快速简单在QQ群中接入ChatGPT的教程。

接入ChatGPT的安装步骤仅需三步，其中分别是：

1. 安装cqhttp
2. 安装nonebot框架
3. 安装chatGPT插件

这里简单说一下各个部分的作用：

-   **go-cqhttp**是基于Onebot协议的一个golang语言的实现。OneBot 是一个聊天机器人应用接口标准，旨在统一不同聊天平台上的机器人应用开发接口。实现了OneBot标准的框架有很多，go-cqhttp只是其中的一种，还有其他的框架例如：[go-cqhttp](https://link.zhihu.com/?target=https%3A//github.com/Mrs4s/go-cqhttp)，[Mirai](https://link.zhihu.com/?target=https%3A//github.com/mamoe/mirai)等等。它们都解析了QQ协议，使用自己的语言实现了统一的接口，**所以可以直白的理解为它们是模拟的一个QQ客户端。**
    此外采用的协议通常是不是手机或者电脑端，**所以登陆cqhttp并不会被挤下去（但是可能接受消息会收到影响）**。
-   第二部分是安装框架，第一步的作用是模拟QQ客户端与QQ服务器进行交流，那么第二部分就是接受这些客户端的消息，并封装这些**发送消息的API接口**，便于我们在**不同的编程语言**中调用。[nonebot](https://v2.nonebot.dev/)则是一个使用python语言实现的机器人框架，同时基于其他语言的框架也有很多，例如TypeScript语言实现的[koishi](https://link.zhihu.com/?target=https%3A//github.com/koishijs/koishi) 框架等等。

最后一部分安装自定义插件，这一部分则是是我们真正需要编写的**机器人的逻辑**，如果需要编写插件则需要学习编程语言，但是如果只是使用别人的插件，**则几行命令就可以搞定**。

所以整个架构大体可以如下图所示

![image-20221208150026750](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208150026750.png)

# 安装cqhttp

## 下载

首先安装cqhttp框架，我们从 [release](https://github.com/Mrs4s/go-cqhttp/releases) 界面下载最新版本的 go-cqhttp，需要根据不同的系统选择不同的文件：

![image-20221208094303224](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208094303224.png)

这里我保存了最常用的系统（Linux，MacOS，Windows 64位）对应的文件，可以[直接下载](https://file.chrisyy.top/go-cqhttp-all.zip)。

系统类型对应更详细的说明：

|    系统类型    |          可执行文件           |            压缩文件             |
| :------------: | :---------------------------: | :-----------------------------: |
| Intel 版 Macos |         Not available         | `go-cqhttp_darwin_amd64.tar.gz` |
|  M1 版 Macos   |         Not available         | `go-cqhttp_darwin_arm64.tar.gz` |
|  32 位 Linux   |         Not available         |  `go-cqhttp_linux_386.tar.gz`   |
|  64 位 Linux   |         Not available         | `go-cqhttp_linux_amd64.tar.gz`  |
|  arm64 Linux   |         Not available         | `go-cqhttp_linux_arm64.tar.gz`  |
|  armv7 Linux   |         Not available         | `go-cqhttp_linux_armv7.tar.gz`  |
| 32 位 Windows  |  `go-cqhttp_windows_386.exe`  |   `go-cqhttp_windows_386.zip`   |
| 64 位 Windows  | `go-cqhttp_windows_amd64.exe` |  `go-cqhttp_windows_amd64.zip`  |
| arm64 Windows  | `go-cqhttp_windows_arm64.exe` |  `go-cqhttp_windows_arm64.zip`  |
| armv7 Windows  | `go-cqhttp_windows_armv7.exe` |  `go-cqhttp_windows_armv7.zip`  |



## 启动

下载完成之后，在Windows下请使用自己熟悉的解压软件自行解压 ，Linux下在命令行中输入 `tar -xzvf [文件名]`进行解压。

在Windows 中**双击打开`go-cqhttp_*.exe`**，根据提示生成运行脚本，随后根据提示操作即可。

在Linux系统中，输入 `./go-cqhttp`, `Enter`运行。

------

之后我们以Linux系统演示后面的步骤，

首次启动时，cqhttp会读取当前目录下是否有`config.yml`文件如果没有则会生成一个**配置文件** 

随后**根据提示选择编号3**（因为nonebot暂时只有反向socket），重新启动后如果成功会显示如下信息

![image-20221208120226802](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208120226802.png)

## 配置

cqhttp启动时会读取当前目录下是否有`config.yml`文件，如果有则会依赖文件中的配置，配置中包含我我们的**账号信息**和**通讯方式**。

我们需要配置的只有两个地方：

1. 修改qq账号
2. 修改`ws-reverse`中`universal`为` ws://127.0.0.1:8080/onebot/v11/ws`最后servers部分的配置如下：

![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208141937474.png)



# 安装NoneBot

第二部安装NoneBot框架，这一步建议使用python中虚拟环境进行安装，并确保python版本在3.8以上。

使用venv创建一个虚拟环境，并使用souce加载环境

```
python3 --version 查看版本
python3 -m venv venv
source venv/bin/activate
```

如果报错请查一下相关资料，如果加载成功可以看到命令行前带有**venv提示符**：

![image-20221208142423327](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208142423327.png)

随后使用pip3安装nonebot脚手架，并通过nb命令创建bot代码，通过`nb run`运行bot具体可以参考[nonebot](https://v2.nonebot.dev/docs/start/installation)文档

```
 pip3 install nb-cli
 pip3 install nonebot-adapter-onebot # 安装适配器
 nb # 生成bot文件
```

![image-20221209140937584](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221209140937584.png)

**请注意！！请注意！！请注意！！**
**在adapter这里【使用空格】来选择OneBot V11适配器**，最后选择完结果如下。

![image-20221208125102633](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208125102633.png)

至此Bot已经基本配置成功，并且安装了一个最简单的内置echo插件（可选的），随后使用`nb run`启动bot或者使用python3 bot来启动

```
 nb run #启动bot
```

要与机器人互动，可以首先使用`/echo`命令让其输出一些信息，斜杠`/`是nonebot默认的命令起始符号，可以自定义设置。

![image-20221208142819097](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208142819097.png)

# 安装插件

最后一步安装我们自定的插件，使用nb直接安装

```
nb plugin install nonebot-plugin-chatgpt
```

安装完成之后，我们需要配置一些信息，在创建机器人根目录下中`.env.dev`文件中填入

```
CHATGPT_SESSION_TOKEN="XXX" # token信息
CHATGPT_COMMAND="chat"			# 触发聊天的命令
CHATGPT_TO_ME="False"				# 是否需要@机器人
```

最后文件显示如下：

![image-20221208160709537](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208160709537.png)

启动bot使用`nb run`命令，可以在输出信息中查看我们的插件是否被加载.

![image-20221208143447455](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208143447455.png)

最后效果如下

![image-20221208143538843](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208143538843.png)

最后，如果不需要斜杠`/`来触发命令，则在`.env.dev`文件中配置如下即可。

```
command_start=[""]
```

那么配置之后的效果则是通过**chat可以直接触发**

![image-20221208162320883](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208162320883.png)

关于上文的插件更多的配置可以参考[仓库链接](https://github.com/A-kirami/nonebot-plugin-chatgpt)

更多的插件请参考[nonebot商店](https://v2.nonebot.dev/store)

# 多账号插件

有兴趣的小伙伴可以加入Fabric闲聊群（然而都在玩ChatGPT）群里有配置好的机器人。

此外群中的机器人并没有使用以上提到的`nonebot-plugin-chatgpt`插件，而是自己开发的一个多`session_token`的`chatGPT`插件，从而满足群中请求量过多导致的API限制问题，使用一下命令安装

```
pip3 install nonebot-plugin-multi-chatgpt --upgrade
```

随后在`bot.py`中加上如下代码，加载插件

```
nonebot.load_plugin('nonebot_plugin_multi_chatgpt')
```

具体配置方法请查[lingyin-bot中的[配置文件](https://github.com/chrisyy2003/lingyin-bot/blob/main/.env.dev#L11-L24)示例，和[插件主页](https://github.com/chrisyy2003/lingyin-bot/tree/main/plugins/chatGPT)

![image-20221211235728170](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221211235728170.png)

## 闲聊群

![image-20221208133611308](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221208133611308.png)
