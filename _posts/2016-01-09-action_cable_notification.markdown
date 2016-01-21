---
title: ActionCable小记
date: 2016-01-09
---
## 前言
ActionCable作为Rails5中新加入的最重要的功能之一，在Rail5正式发布前就已经引人瞩目。
ActionCable通过使用WebSocket协议，可以实现从Server端主动向Client端推送消息。
同时也在Client端建立了通向Server端的连接，以保证消息推送的安全性和可靠性。  
前一段时间我曾用ActionCable实现了简单的Realtime Notification推送功能和一个简单的chatroom。
Rails5 beta发布时，DHH也曾分享了一个关于ActionCable的一个demo视频。
好的东西需要去分享，我也向大家介绍下我在使用ActionCable时的一些心得。

## 环境搭建
ActionCable
