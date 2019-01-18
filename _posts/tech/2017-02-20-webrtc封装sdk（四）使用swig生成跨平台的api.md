---
layout: post
title: webrtc封装sdk（四）使用swig生成跨平台的api
category: 技术
tags: PHP
description: webrtc封装sdk（四）使用swig生成跨平台的api
---


## WebRtc中VoiceEngine的使用方法 ##

webrtc中的VoiceEngine是用来管理语音通道channel的类
提供了控制语音整个过程的接口
VoiceEngine的实现类VoiceEngineImpl通过继承的方式将很多不同类型的接口集成在了一个类对象中。
这些接口一共分为以下几种类型：

 - **VoiceEngine** ：基础接口，可以设置log文件路径，创建VoiceEngineImpl对象
 - **VoEAudioProcessingImpl** ：提供了语音信号处理的功能，如噪声抑制Noise Suppression)、自动增益控制AGC、回声消除EC等功能
 - **VoECodecImpl** ：提供了编解码设置、FEC保护、码率设置等功能
 - **VoEExternalMediaImpl** ：实现对语音数据预处理的回调函数设置
 - **VoEFileImpl** ：提供了对语音数据录音、播放语音文件作为语音输入的功能
 - **VoEHardwareImpl** ：提供了对语音设备管理的功能
 - **VoENetEqStatsImpl** ：提供了获取语音neteq模块工作状态信息的功能
 - **VoENetworkImpl** ：提供了语音数据的接收和发送的功能
 - **VoERTP_RTCPImpl** ：提供了rtp/rtcp协议控制的接口，如设置本地ssrc、nack状态、获取rtp/rtcp统计信息等功能
 - **VoEVideoSyncImpl** ：提供了音视频同步播放的功能
 - **VoEVolumeControlImpl** ：提供了对采集和播放声音的控制功能
 - **VoEBaseImpl** ：提供了channel创建，接收发送的开关、播放开关等功能

以上所有基类提供的接口最终的实现其实都是基于Channel这个类
Channel类提供了以上所有基类的接口的具体实现，所以Channel类非常复杂，代码量也很大
其他类只是通过一个channelId找到要操作的channel对象，然后调用channel的接口来实现

基本的调用流程都是如下形式：

      voe::ChannelOwner ch = _shared->channel_manager().GetChannel(channel);
      voe::Channel* channelPtr = ch.channel();
      if (channelPtr == NULL) {
        _shared->SetLastError(VE_CHANNEL_NOT_VALID, kTraceError,
                              "SetRTCPStatus() failed to locate channel");
        return -1;
      }
      channelPtr->SetRTCPStatus(enable);

所以在理解voiceEngine时，一定要理解channel的概念，channel可以用来发送本地数据，也可以用来接收远端数据
其实在[前一篇博客][1]中提到的AudioSendStream AudioReceiveStream也是变相的操作着channel
# 创建VoiceEngine
创建VoiceEngine非常简单

    VoiceEngine* voe = VoiceEngine::Create();

# 获取各种接口

    VoECodec* codec = VoECodec::GetInterface(VOE.ENGINE);
    VoEBase* base = VoEBase::GetInterface(VOE.ENGINE);
    VoEVolumeControl* volumeControl = VoEVolumeControl::GetInterface(VOE.ENGINE);
    VoEHardware* hardware = VoEHardware::GetInterface(VOE.ENGINE);
    VoERTP_RTCP* rtpRtcp = VoERTP_RTCP::GetInterface(VOE.ENGINE);
    VoEAudioProcessing* audioProcessing = VoEAudioProcessing::GetInterface(VOE.ENGINE);
    VoEExternalMedia* externalMedia = VoEExternalMedia::GetInterface(VOE.ENGINE);

# 共享资源SharedData
SharedData提供了在使用webrtc语音引擎时，只需要创建一次的资源，主要有以下几种：

 - 管理channel对象的ChannelManager   
 - 管理音频硬件设备的AudioDeviceModule,比如设置音量控制，选取采集设备
 - 对采集数据的预处理TransmitMixer,比如可以对采集的数据录音
 - 对接收数据播放前的处理OutputMixer，比如可以对接收的数据录音
 - 定时任务处理的线程ProcessThread

 
 


  [1]: http://www.jianshu.com/p/f4856c68f9fd

关键词：webrtc voice engine api 语音引擎