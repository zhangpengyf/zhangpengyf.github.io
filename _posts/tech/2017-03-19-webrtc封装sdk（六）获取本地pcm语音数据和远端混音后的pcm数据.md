---
layout: post
title: webrtc封装sdk（六）获取本地pcm语音数据和远端混音后的pcm数据
category: 技术
tags: webrtc
description: webrtc封装sdk（六）获取本地pcm语音数据和远端混音后的pcm数据
---

做录音时首先需要获取本地采集的pcm数据和所有远端用户合成后的pcm数据，也就是播放时投递给扬声器的pcm数据，本文讲解一下如何获取webrtc的原始音频数据。
## webrtc版本说明
本文使用的webrtc api依赖于webrtc分支版本<=branch57
在branch57以及以前的版本应该都能测试通过。
在branch>=58中VoEExternalMedia可能会被移除。
## 使用的接口
使用的接口为webrtc::VoEExternalMedia类
所在文件为：webrtc/voice_engine/include/voe_external_media.h
首先需要实现一个数据回调使用的callback类：

    class AudioMixDataCallBack :public webrtc::VoEMediaProcess
    {
    public:
        virtual void Process(int channel,
                             webrtc::ProcessingTypes type,
                             int16_t audio10ms[],
                             size_t length,
                             int samplingFreq,
                             bool isStereo)
        {
            if (type == webrtc::kPlaybackAllChannelsMixed)
            {
                printf("get remote mix pcm data\n");
            }
            if (type == webrtc::kRecordingAllChannelsMixed)
            {
                printf("get local record pcm data\n");
                //本地声音是连续的，如果做录音mp3应该以本地回调为时间参考
                //在本地回调时录音
            }
        }
    };

## 如何拿到数据
开启webrtc数据回调：

      AudioMixDataCallBack* p = new AudioMixDataCallBack();
        webrtc::VoEExternalMedia* externalMedia =
                    webrtc::VoEExternalMedia::GetInterface(g_voe->engine());
        //回调本地录制数据
        externalMedia->RegisterExternalMediaProcessing(-1, 
                    webrtc::kRecordingAllChannelsMixed, 
                    *p);
        //回调所有远端的合成数据
        externalMedia->RegisterExternalMediaProcessing(-1, 
                    webrtc::kPlaybackAllChannelsMixed, 
                    *p);
        return 0;

注意拿到混合后的数据要求RegisterExternalMediaProcessing的第一个参数channelId设置为-1

在webrtc branch55,branch56中根据以上方法就可以拿到数据了，但是在branch57中需要修改一下webrtc的代码才能拿到数据。
修改的文件为：webrtc/audio/audio_state.cc

    AudioState::AudioState(const AudioState::Config& config)
        : config_(config),
          voe_base_(config.voice_engine),
          audio_transport_proxy_(voe_base_->audio_transport(),
                                 voe_base_->audio_processing(),
                                 config_.audio_mixer) {
      process_thread_checker_.DetachFromThread();
      RTC_DCHECK(config_.audio_mixer);
    
      // Only one AudioState should be created per VoiceEngine.
      RTC_CHECK(voe_base_->RegisterVoiceEngineObserver(*this) != -1);
    
      auto* const device = voe_base_->audio_device_module();
      RTC_DCHECK(device);
    
      // This is needed for the Chrome implementation of RegisterAudioCallback.
      //注释下面两行代码
      // device->RegisterAudioCallback(nullptr);
      // device->RegisterAudioCallback(&audio_transport_proxy_);
    }
    
## 正确的调用堆栈
branch57中如果不注释以上两行代码，会导致播放线程不会到VoEBaseImpl::NeedMorePlayData()拿播放数据，导致我们设置的回调不会执行。
修改后audio render线程在播放声音之前会把混音后的数据回调到我们定义的AudioMixDataCallBack中
调用堆栈为：

    AudioMixDataCallBack::Process()
    AudioMixDataCallBack::Process()
    webrtc::voe::OutputMixer::DoOperationsOnCombinedSignal()
    webrtc::VoEBaseImpl::GetPlayoutData()
    webrtc::VoEBaseImpl::NeedMorePlayData()
    webrtc::VoEBaseImpl::NeedMorePlayData()
    webrtc::AudioDeviceBuffer::RequestPlayoutData()
    webrtc::AudioDeviceMac::RenderWorkerThread()
    webrtc-test`webrtc::AudioDeviceMac::RunRender()
    
## 后续工作
据说VoEExternalMedia类将会在新版webrtc中删除，在最新版本的webrtc中如何拿到数据目前还没有研究。

关键词：webrtc 录音 pcm数据 audio 混音