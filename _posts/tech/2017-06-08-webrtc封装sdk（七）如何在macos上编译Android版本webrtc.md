---
layout: post
title: webrtc封装sdk（七）如何在macos上编译Android版本webrtc
category: 技术
tags: webrtc
description: webrtc封装sdk（七）如何在macos上编译Android版本webrtc
---

有些开发的朋友习惯使用macos来工作，所以需要在macos上编译webrtc android版本，
但是根据webrtc官方的说法，目前只支持在linux系统下编译webrtc android版本，
经过自己的研究发现，其实可以通过很少的修改，在mac上编译通过webrtc android。
这里说的方法不是使用linux虚拟机，是真的在macos下编译。
下面讲解一下如果操作。
## 准备阶段
你需要下载两份webrtc代码
一份是mac系统下的mac或ios版本
一份是linux系统下的android版本

## 1. 下载mac系统下的android ndk/sdk
首先打开webrtc中的文件build/config/android/config.gni
此文件写明了应该使用哪个版本的sdk，ndk，以及它们的引用路径。
我们需要自己下载对应版本ndk,sdk，并且需要修改此文件中的引用路径，来告诉webrtc编译时使用我们下载的sdk,ndk。
###ndk版本为r12b：

    if (!defined(default_android_ndk_root)) {
        default_android_ndk_root = "//third_party/android_tools/ndk"
        default_android_ndk_version = "r12b"
        default_android_ndk_major_version = 12
      } else {
        assert(defined(default_android_ndk_version))
        assert(defined(default_android_ndk_major_version))
      }
下载地址为：https://dl.google.com/android/repository/android-ndk-r12b-darwin-x86_64.zip
其他版本把r12b替换为你要的版本即可
旧版本ndk如何下载？参考:[如何下载旧版ndk][1]
### sdk版本为25(with Google Play services)：

    if (!defined(default_android_sdk_root)) {
        default_android_sdk_root = "//third_party/android_tools/sdk"
        default_android_sdk_version = "25"
        default_android_sdk_build_tools_version = "25.0.2"
      }
      if (!defined(default_lint_android_sdk_root)) {
        # Purposefully repeated so that downstream can change
        # default_android_sdk_root without changing lint version.
        default_lint_android_sdk_root = "//third_party/android_tools/sdk"
        default_lint_android_sdk_version = "25"
      }
      if (!defined(default_extras_android_sdk_root)) {
        # Purposefully repeated so that downstream can change
        # default_android_sdk_root without changing where we load the SDK extras
        # from. (Google Play services, etc.)
        default_extras_android_sdk_root = "//third_party/android_tools/sdk"
      }

下载sdk可以通过android studio中的sdk管理器下载带Google Play services的sdk

##2.执行官方的编译命令，发现错误拷贝缺失文件
现在我们尝试按照官网说明来编译android
执行如下命令：

    gn gen out/Debug --args='target_os="android" target_cpu="arm"'
    ninja -C out/Debug
此时会发生错误，提示缺少文件
我们需要把linux系统下下载的webrtc android目录下对应的文件拷贝过来。
这里可能有大概10个左右的缺失文件或目录，需要每个都拷贝过来。
当拷贝完成后，就解决了mac下编译webrtc android的问题。

  [1]: http://www.jianshu.com/p/b50aaae78140