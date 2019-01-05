---
layout: post
title: webrtc封装sdk（八）使用webrtc desktop_capture截屏
category: 技术
tags: webrtc
description: webrtc封装sdk（八）使用webrtc desktop_capture截屏
---

### 简介
webrtc的modules中有一个模块desktop_capture，该模块负责截屏，目前只支持windows和mac平台，android,ios没有实现。

desktop_capture中有两种截屏方式，第一种是截单个窗口，叫做WindowCapturer，
第二种是截整个屏幕，叫做ScreenCapturer。
window_capture/screen_capture都继承于基类DesktopCapturer：

    // Abstract interface for screen and window capturers.
    class DesktopCapturer {
     public:
     //初始化截屏，设置数据回调
      virtual void Start(Callback* callback) = 0;
      //截一张图，数据直接进入callback
      virtual void Capture(const DesktopRegion& region) = 0;
    };

### 一、WindowCapture
WindowCapture主要增加了获取窗口列表，和设置截屏窗口id的接口：

      virtual bool GetWindowList(WindowList* windows);
      virtual bool SelectWindow(WindowId id);

### 二、ScreenCapture
ScreenCapture主要增加了获取屏幕列表，和设置截屏屏幕id的接口：

      virtual bool GetScreenList(ScreenList* screens);
      virtual bool SelectScreen(ScreenId id);

### 三、使用流程
接口都比较简单，很容易使用，大概的流程如下：
 1. 创建对象
 2. 初始化截屏，设置回调函数
 3. 开启线程循环截图


    screen_capture_ = webrtc::ScreenCapturer::Create(webrtc::DesktopCaptureOptions::CreateDefault());
    screen_capture_->SelectScreen(0);
    
    bool ImageCaptureThreadFunc(void* param)
    {
        webrtc::DesktopCapturer* capture = static_cast<webrtc::DesktopCapturer*>(param);
        capture->Capture(webrtc::DesktopRegion(webrtc::DesktopRect()));
        Sleep(100);
        return true;
    }

### 四、截屏数据处理
截屏后得到的数据格式是rgb，需要使用libyuv将数据从rgb转换为yuv420，然后传入编码器和进行本地渲染。
转换时注意填写正确的原始数据类型，windows下格式为webrtc::kARGB

    void OnCaptureCompleted(webrtc::DesktopFrame* frame) {
    
    	if (frame == NULL) {
    		//error,stop capture
    		StopImageCapture();
    		return;
    	}
    	int width = frame->size().width();
    	int height = frame->size().height();
    	int half_width = (width + 1) / 2;
    
    	webrtc::VideoFrame i420frame;
    	i420frame.CreateEmptyFrame(width, height, width, half_width, half_width);
    
    	int ret = webrtc::ConvertToI420(webrtc::kARGB, frame->data(), 0, 0, width, height, 4, webrtc::kVideoRotation_0, &i420frame);
    	if (ret != 0) {
    		return;
    	}
    
        //后续处理
    	OnIncomingCapturedFrame(0, i420frame);
        //需要释放内存
    	delete frame;
    }

### 五、传递给videoSendStream
通过VideoSendStream的input接口可以把采集到的图像投递进去，编码发送。

    video_stream_s_->Input()->IncomingCapturedFrame(videoFrame);