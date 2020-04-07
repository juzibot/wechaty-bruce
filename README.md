---
title: "社区的完美助手-wechaty机器人"
date: 2020-04-07 20:00 +0800
author: bruce
categories: tutorial
tags:
  - wechaty
  - financial
---
<!-- markdownlint-disable -->

> 作者:bruce，IT 后端从业者（php、golang）
<!-- more -->

首先，在这里我给大家说一声对不起。对于node.js也是现学现用，因此有各种不规范，欢迎各位指出。

作为一个从互联网的程序猿，在传统的行业中作业时，会有一些需求过于人工化。那么我今天将我使用Wechaty 的 实现一款社群管理机器人分享给大家。

## 我在做一件自动化的事情

在微信社群管理中，我们时长需要人工去手动处理一系列事情，我需要做这么一些事情：

- 自动通过好友验证
- 私聊关键字回复
- 自动聊天
- 自动邀请加入群聊自动欢迎


每天都要花费不少的时间做这一系列事情，如果能用机器实现自动化，那么我可以节省非常多的时间。加上现有的绝大部分客户都落地在微信号中，那么第一个思路就是：如何做一个微信个人号的机器人。经过多方搜索，终于找到了Wechaty这个个人号框架。

## 机器人计划实现的功能

- [x] 自动通过好友验证
	- [x] 当有人添加机器人时，判断验证消息关键字后通过或直接通过
	- [x] 通过验证后自动回复并介绍机器人功能
- [x] 私聊关键字回复
	- [ ] 例如回复 加群 推送群聊邀请
	- [ ] 例如回复 作者微信 推送作者微信名片
- [x] 自动聊天
	- [x] 群聊中通过 @[机器人]xxx 可以和机器人聊天
	- [x] 私聊发送消息即可聊天
- [ ] 加入群聊自动欢迎
	- [x] 当新的小伙伴加入群聊后自动 @[新的小伙伴] 发一个文字欢迎

## 代码实现的思路

功能比较简单，分多个就是为了不让所有代码都在一个文件，简单分开下

|-- src/
|---- index.js				# 入口文件
|---- config.js		  	# 配置文件
|---- onScan.js				# 机器人需要扫描二维码时监听回调
|---- onRoomJoin.js 	# 进入房间监听回调
|---- onMessage.js		# 消息监听回调
|---- onFriendShip.js	# 好友添加监听回调
|-- package.json

初期准备
首先新建文件夹，初始化

npm init -y
接着我们安装比较重要的核心包

// Wechaty核心包
npm install --save wechaty
// padplus协议包
npm install --save wechaty-puppet-padplus
我们在开发过程中，还需要用到qrcode-terminal这个包，作用就是将二维码输出在终端来供我们扫码登录

npm install --save qrcode-terminal
然后就可以愉快的开发了，没错就是这么简单

配置文件
所谓的配置文件，就是那个 config.js ，只是把我们需要用到的一些可配置参数拿出来

module.exports = {
  // puppet_padplus Token
  token: "你自己申请的ipad协议token",
  // 你的机器人名字
  name: "圈子",
  // 房间/群聊
  room: {
    // 管理群组列表
    roomList: {
      // 群名字(用于发送群名字加群):群id，后面会介绍到
      Web圈: "*****@chatroom",
      男神群: "*****@chatroom"
    },
    // 加入房间回复
    roomJoinReply: `你好，欢迎加入`
  },
  // 私人
  personal: {
    // 好友验证自动通过关键字
    addFriendKeywords: ["加群", "前端"],
    // 是否开启加群
    addRoom: true
  }
}

入口文件
入口文件，也就是我们 src 目录下的 index.js 文件

这里做的很简单，没有逻辑

首先引入我们包

const { Wechaty } = require("wechaty") // Wechaty核心包
const { PuppetPadplus } = require("wechaty-puppet-padplus") // padplus协议包
const config = require("./config") // 配置文件
接着初始化我们的bot

// 初始化
const bot = new Wechaty({
  puppet: new PuppetPadplus({
    token: config.token
  }),
  name: config.name
})
接下来一段链式调用，监听，启动，完事

const onScan = require("./onScan")
const onRoomJoin = require("./onRoomJoin")
const onMessage = require("./onMessage")
const onFriendShip = require("./onFriendShip")
bot
  .on("scan", onScan) // 机器人需要扫描二维码时监听
  .on("room-join", onRoomJoin) // 加入房间监听
  .on("message", onMessage(bot)) // 消息监听
  .on("friendship", onFriendShip) // 好友添加监听
  .start()

onScan
onScan 文件是我们在机器人需要扫描二维码时的监听回调

这里面的代码超级简单

const Qrterminal = require("qrcode-terminal")
module.exports = function onScan(qrcode, status) {
  Qrterminal.generate(qrcode, { small: true })
}
首先引入 qrcode-terminal 包

这个回调中其实做的很简单，回调接收了两个参数

qrcode qr码
status 状态
我们借助Qrterminal.generate这个API将 qr 码输出到终端而已，后面那个small参数是因为qrcode-terminal 这个包默认输出的二维码太大了，给它变小一些

onFriendShip
onFriendShip是friendship事件监听的回调，好友添加监听

const { Friendship } = require("wechaty")
// 配置文件
const config = require("./config")
// 好友添加验证消息自动同意关键字数组
const addFriendKeywords = config.personal.addFriendKeywords
// 好友添加监听回调
module.exports = async function onFriendShip(friendship) {
  let logMsg
  try {
    logMsg = "添加好友" + friendship.contact().name()
    console.log(logMsg)
    switch (friendship.type()) {
      /**
       * 1. 新的好友请求
       * 设置请求后，我们可以从request.hello中获得验证消息,
       * 并通过`request.accept（）`接受此请求
       */
      case Friendship.Type.Receive:
        // 判断配置信息中是否存在该验证消息
        if (addFriendKeywords.some(v => v == friendship.hello())) {
          logMsg = `自动通过验证，因为验证消息是"${friendship.hello()}"`
          // 通过验证
          await friendship.accept()
        } else {
          logMsg = "不自动通过，因为验证消息是: " + friendship.hello()
        }
        break
      /**
       * 2. 友谊确认
       */
      case Friendship.Type.Confirm:
        logMsg = "friend ship confirmed with " + friendship.contact().name()
        break
    }
    console.log(logMsg)
  } catch (e) {
    logMsg = e.message
  }
}
如上所示，我们想加好友时，验证消息填写我们指定的文字可以自动通过



```

通过这几个核心模块的完成，第一版要搭建的核心骨架和功能就完成了，

## 运行效果

| 我 | 机器人 |
|:--|:--|
| hi |  |
|  | 您好，干嘛呀。 |
|  | 那么，还是请跟随我来吧。 |
|  | - 发送[可转债]了解转债信息。- 发送[后端]加入技术交流群- 。 |

## 后续计划和致谢

首先，感谢开源项目[Wechaty](https://github.com/wechaty/wechaty)团队以及其提供的开发者计划，让我有机会能实现自己的想法。

目前的机器人还只能做一些简单的事情，后续会根据业务的需求增加更多的功能上去。目标是让机器人为我打工。
