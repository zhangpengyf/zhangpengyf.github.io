---
layout: post
title:  "如何编译webrtc ios"
date:   2015-10-10 19:26:00
categories: 音视频
excerpt: webrtc
---

* content
{:toc}




###使用ninja编译

 export GYP_CROSSCOMPILE=1
 export GYP_DEFINES="OS=ios target_arch=arm"
 export GYP_GENERATOR_FLAGS="output_dir=out_ios"
 export GYP_GENERATORS=ninja
 cd src/
 webrtc/build/gyp_webrtc
 ninja -C out_ios/Debug-iphoneos AppRTCDemo


###使用xcode编译

 export GYP_DEFINES="OS=ios"
 export GYP_GENERATOR_FLAGS="xcode_project_version=3.2 xcode_ninja_target_pattern=All_iOS  xcode_ninja_executable_target_pattern=AppRTCDemo|libjingle_peerconnection_unittest|libjingle_peerconnection_objc_test output_dir=out_ios"
export GYP_GENERATORS="ninja,xcode-ninja"
 cd src
 webrtc/build/gyp_webrtc
