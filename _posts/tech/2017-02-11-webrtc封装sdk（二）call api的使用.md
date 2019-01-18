---
layout: post
title: webrtc封装sdk（二）call api的使用
category: 技术
tags: PHP
description: webrtc封装sdk（二）call api的使用
---

## 为什么使用Call Api
目前新版webrtc的api和以前已经有很大不同，如果希望封装webrtc为一个音视频sdk，
目前最简单的方法就是了解并使用call类的api。

本文简单介绍Webrtc::Call的基本使用方法。

文中提到的代码可以参考我的开源项目：

- [Mac端可以运行的call调用demo](https://github.com/zhangpengyf/webrtc-native-example)

- [foxrtc--基于webrtc封装的sdk](https://github.com/zhangpengyf/foxrtc)

## Call简介
Call类的头文件为webrtc/call.h

Call类的基本功能为管理rtp媒体流，负责整个音视频通话的管理。

使用Call类的好处是，Call将会回调给你原始的rtp/rtcp数据，并且能将收到的网络rtp/rtcp数据流直接传给Call进行处理。

这样避免了去研究webrtc中的会话管理的协议部分，如PeerConnection类，减轻了研究的复杂度。

Call主要负责管理四种数据流（即Stream）：

- VideoSendStream：负责产生本地视频数据

- VideoReceiveStream：负责处理远端某一个端的视频数据

- AudioSendStream：负责产生本地音频数据

- AudioReceiveStream：负责接收远端某一个端的语音数据

## 下面演示一下call的创建方法

在初始化音视频sdk时需要创建一个call对象

```cpp

Call::Config callConfig;
gVoe = VoiceEngine::Create();
webrtc::AudioState::Config audioStateConfig;
audioStateConfig.voice_engine = gVoe;
callConfig.audio_state = AudioState::Create(audioStateConfig);
callConfig.bitrate_config.max_bitrate_bps = 500*1000;
callConfig.bitrate_config.min_bitrate_bps = 100*1000;
callConfig.bitrate_config.start_bitrate_bps = 250*1000;
g_call = Call::Create(callConfig); 

```

我们可以通过VoiceEngie的实例gVoe来对声音进行各种操作，创建完Call的实例就可以开始创建4种Stream了。

## VideoSendStream
一般创建一个VideoSendStream即可，发送本地一个视频流

首先需要配置视频编码参数，创建编码器
```cpp
      VideoSendStream::Config streamConfig(g_transport);
      streamConfig.encoder_settings.payload_name = "VP9";
      streamConfig.encoder_settings.payload_type = PT_VP9;
      streamConfig.rtp.max_packet_size = 1350;
      streamConfig.encoder_settings.encoder =
      webrtc::VideoEncoder::Create(VideoEncoder::kVp9);
      VideoEncoderConfig encodeConfig;
      VideoSendStream* stream = g_call->CreateVideoSendStream(
      std::move(streamConfig), std::move(encodeConfig));
```
## VideoReceiveStream
多人会话中，需要根据远端的ssrc来创建多个对应的VideoReceiveStream
对某一个远端视频流的管理通过ssrc对应的VideoReceiveStream接口来操作

```cpp
   
      VideoReceiveStreamInfo* info = new VideoReceiveStreamInfo(id, remoteSsrc);
      VideoReceiveStream::Config streamConfig(g_transport);
      VideoReceiveStream* stream = g_call->CreateVideoReceiveStream(std::move(streamConfig));
```

## AudioSendStream
一般创建一个AudioSendStream即可，发送本地一个语音流
需要先使用VoiceEngine创建一个channel,后续也可以根据channel id通过g_Voe提供的接口操作这个通道

      AudioSendStream::Config streamConfig(g_transport);
      streamConfig.voe_channel_id = VoEBase::GetInterface(g_Voe)->CreateChannel();
      streamConfig.rtp.ssrc = ssrc;
      AudioSendStream* stream =
      g_call->CreateAudioSendStream(std::move(streamConfig));

## AudioReceiveStream
多人会话中，需要根据远端的ssrc来创建多个对应的AudioReceiveStream
对某一个远端语音流的管理通过ssrc对应的AudioReceiveStream接口来操作

	  rtc::scoped_refptr<webrtc::AudioDecoderFactory> g_audioDecoderFactory = CreateBuiltinAudioDecoderFactory();
      AudioReceiveStream::Config streamConfig;
      streamConfig.rtp.local_ssrc = localSsrc;
      streamConfig.rtp.remote_ssrc = remoteSsrc;
      streamConfig.rtcp_send_transport = &info->transport;
      streamConfig.voe_channel_id = VoEBase::GetInterface(g_Voe)->CreateChannel();
      streamConfig.decoder_factory = g_audioDecoderFactory;
      AudioReceiveStream* stream =
      g_call->CreateAudioReceiveStream(std::move(streamConfig));

## Transport
代码中出现的g_transport是Transport类的具体实现
通过该对象，创建的所有stream可以回调要发出的rtp/rtcp包
你需要自己实现这个对象，完成将数据流传输到媒体服务器的功能

    class Transport {
     public:
      virtual bool SendRtp(const uint8_t* packet,
       size_t length,
       const PacketOptions& options) = 0;
      virtual bool SendRtcp(const uint8_t* packet, size_t length) = 0;
    
     protected:
      virtual ~Transport() {}
    };

## 接收远端rtp/rtcp数据

接收到的远端数据直接传递给Call对象即可

    PacketTime pt;
    g_call->Receiver()->DeliverPacket(MediaType::ANY, (const uint8_t*)data, len,
      pt);

## 视频的采集和渲染
VideoSendStream的原始图像是通过VideoCaptureModule采集到的

VideoSendStreaam本身不负责采集，只负责编码和发送视频帧

所以需要自己去开启采集功能

视频的渲染也不由VideoReceiveSteam负责，它会将解码后的videoFrame回调上来

可以参考rtc::VideoSourceInterface类和rtc::VideoSourceInterface类

关键词：webrtc native api c++ call stream