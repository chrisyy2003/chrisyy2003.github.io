---
layout: post
title: 5分钟在QQ群搭建ChatGPT机器人！
date: 2022-12-07 23:35:05
categories:
    - other
---



>   说明：问题在于chatgpt插件出现问题后不能及时更新，所以文章中插件部分可能会**出现过时问题导致不能用的情况**。但是cqhttp和nonebot**部分的安装教程不会有问题**，**所以依然可以作为参考进行安装**。
>
>   有问题的都是针对插件。
>
>   此外，openai现在高强度更新antibot策略可能策略随时会变，建议没有基础的朋友先等待，有基础的可以关注最新方案尝鲜。

# 12月14日更新

1.  文章提到的nonebot-plugin-chatgpt 插件以及合并了绕过cf的方法，可以继续按照本文安装。
2.  关于群中的多账号bot可以直接通过修改[token.json](https://github.com/chrisyy2003/lingyin-bot/blob/main/chatgpt_token.json)来完成自己的配置。几个账号就放几个token。

这两个方法是不同的，第一个是一个插件，第二个是一个群中专用的集成的bot。

# 12月13日更新

>   现在在有GUI（图形界面）的机器上可以实现半自动的方法，不过还需要修改插件，有基础的可以自己先试一试。
>

1.  在桌面环境中：`pip3 install revChatGPT==0.0.42.1` 安装最新版的revChatGPT，此外还需要安装谷歌浏览器
2.  随后在test.py文件中【**只**】配置session_token，像下面这样。

```
{
  "session_token": "<YOUR_TOKEN>",
}
```

最后test文件如下

```
from revChatGPT.revChatGPT import Chatbot

config = {
    "session_token": "XX", ##token
    "proxy": "http://127.0.0.1:6152"
}
chatbot = Chatbot(config, conversation_id=None)

res = chatbot.get_chat_response("hello")
print(res)
```

程序会弹出浏览器自动获取cf值，能够跑过的话接下来：

1.  直接安装lingyin-bot
2.  配置env.dev

安装方式在主页。（因为这个集成的bot直接依赖这个库的，所以test.py可以，bot就可以）



# ~~openai上了Cloudflare的临时解决方案~~

目前情况是寄了，暂时还没有一个很好的解决办法。

**此方案是可以成功的但是非常麻烦**，群友反馈结果是限制比较大，**需要手动换cf字段值并且cf有效时间只有2小时**，而且调用次数过多就寄，暂时不推荐此方案了。

可以等等，开发者正在解决问题中，有问题请查看revChatGPT仓库最新commit进行解决。

加discord关注最新进展，https://discord.gg/RvBkdcYUrH

## 安装步骤

1.  首先下载lingyin-bot文件，或者clone
    ![image-20221212171748432](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212171748432.png)
2.  更新revchatGPT，`pip3 install revChatGPT --upgrade`
3.  复制图中的cf_clearance的值
    ![image-20221212103154911](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212103154911.png)
4.  拿到user agent值，User-agent的值看这个图
    ![image-20221212124926694](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212124926694.png)
5.  然后在 https://github.com/chrisyy2003/lingyin-bot/blob/main/test.py#L8-L9 修改为自己的token和cf cookie值和user agent值， 并随后运行[test.py](https://github.com/chrisyy2003/lingyin-bot/blob/main/test.py)文件，这是来检查你的信息能够正确的请求，如果test能跑过那么根据bot[主页](https://github.com/chrisyy2003/lingyin-bot)的说明安装后，bot就也能跑了。
    跑过指的是能返回如下的有效消息：
    ![image-20221212103356692](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212103356692.png)
    如果遇见下面的问题
    ![image-20221212150847725](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212150847725.png)
    **请确保获取的cf的设备和bot是同一台设备！！！**就是你本地获取的cf就只能本地跑，本地test能过，放到服务器上是不行的
6.  test能过之后就配置bot的env.dev如下图![](https://chrisyy-images.oss-cn-chengdu.aliyuncs.com/img/image-20221212123451926.png)
7.  随后启动bot，启动方法请看lingyin-bot主页

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

>   windows中可能不是用python3，而是直接用python，没有生成venv文件夹请看python指令是否正确。
>
>   同理pip3在windows直接是pip

使用venv创建一个虚拟环境，并使用souce加载环境

```
python3 --version 查看版本
python3 -m venv venv
source venv/bin/activate
```

**windows下请直接在命令行启动activate例如**

```
// powershell
./activate.ps1

// cmd命令行
./activate.bat
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
