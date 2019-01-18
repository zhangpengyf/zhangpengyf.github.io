---
layout: post
title: webrtc封装sdk（一）思路
category: 技术
tags: PHP
description: webrtc封装sdk（一）思路
---

### 目的
很多公司使用webrtc来做音视频sdk，但是大部分公司在使用上层的api，使用起来很繁琐，需要了解很多会话协议，《webrtc封装sdk》系列文章为大家讲述一种更简单的封装方法，只需几天，就可以封装出一个sdk。

### 为何如此简单？
本文讲述的方法，不处理会话管理部分的逻辑，只针对音视频数据包，通过使用webrtc内部的c++接口来实现音视频的基本功能，并且能够回调上来原始的rtp/rtcp数据包，也可以传入接收到的原始数据包，操作起来非常简单。

### 实现思路
通过使用webrtc/call.h中的webrtc::Call类来创建本地和远端视频/音视频流，产生和接收rtp/rtcp数据包。

call类包括的概念大概有以下几种：

- VideoSendStream：负责产生本地视频数据

- VideoReceiveStream：负责处理远端某一个端的视频数据

- AudioSendStream：负责产生本地音频数据

- AudioReceiveStream：负责接收远端某一个端的语音数据

如果你也需要封装webrtc来做音视频sdk，但是对最新代码又不是很了解，可以来参考我的开源项目：

1、https://github.com/zhangpengyf/webrtc-native-example ：mac端可以运行的demo

master分支演示了如何使用call接口来封装sdk，getaudiodata分支，演示了如何拿到本地采集音频数据，和远端混音后的原始数据，通过这些数据可以进行录音

2、https://github.com/zhangpengyf/foxrtc : 封装webrtc为一个音视频sdk [开发中]

通过参考我的思路和demo你会发现原来封装sdk还可以如此简单。

### 编译技巧
由于只需要使用call api我们只需要编译call这个target，这样可以节省三分之二的编译时间。

编译命令为：

gn gen out/mac_Debug --ide=xcode --args='is_debug=false  rtc_enable_protobuf=false  rtc_include_tests=false'

ninja -C out/mac_Debug call



关键词：webrtc 封装 sdk webrtc c++ api webrtc call