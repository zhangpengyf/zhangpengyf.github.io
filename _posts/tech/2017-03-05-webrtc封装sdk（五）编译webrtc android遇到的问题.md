---
layout: post
title: webrtc封装sdk（五）编译webrtc android遇到的问题
category: 技术
tags: webrtc
description: webrtc封装sdk（五）编译webrtc android遇到的问题
---

按照官方的编译步骤就可以编译出android版本的各个静态库libxxx.a
当我们使用这些静态库，并且还需要编译自己写的那些c++代码时，可能会遇到以下两个问题

 - **自己本地的android ndk和webrtc内部使用的ndk版本不同**

 - **ndk版本相同但是stl的libc++库类型不同，如llvm-libc++,gnustl,stlport等**

以上两个问题会导致如下类型的链接错误：

    undefined reference to 'std::__ndk1::locale::~locale()'
    undefined reference to 'std::__ndk1::ctype<char>::id'
    undefined reference to 'std::__ndk1::ios_base::clear(unsigned int)'
    undefined reference to 'std::__ndk1::ios_base::getloc() const'
如果需要查找到底哪个libstdc++.a含有这写符号，可以ndk目录下批量nm|grep一下看看。

## 下面讲解一下如何解决上面两个问题。

## 1、和webrtc使用相同版本的ndk
   仔细阅读webrtc源码中的文件build/config/android/config.gni，可以发现webrtc默认使用的ndk版本和sdk版本为r12b：
    
        if (!defined(default_android_ndk_root)) {
        default_android_ndk_root = "//third_party/android_tools/ndk"
        default_android_ndk_version = "r12b"
        default_android_ndk_major_version = 12
      } else {
        assert(defined(default_android_ndk_version))
        assert(defined(default_android_ndk_major_version))
      }
      
  并且可以发现webrtc使用的是llvm-libc++类型的c++库为：
   
    android_libcpp_root = "$android_ndk_root/sources/cxx-stl/llvm-libc++"
     
我们根据这个版本去下载一个同样版本的ndk:  https://dl.google.com/android/repository/android-ndk-r12-darwin-x86_64.zip
安装到系统或者直接使用webrtc中的ndk。

进入ndk目录，并且执行以下命令创建standalone toolchain，重点关注参数--stl=libc++

参数也可以取其它值，可以阅读以下make-standalone-toolchain.sh脚本的内容。
    
    ./build/tools/make-standalone-toolchain.sh --platform=android-16 \
    --install-dir=./my-android-toolchain --stl=libc++
     
     
把生成的my-android-toolchain拷贝到~/my-android-toolchain，
这样就生成了和webrtc相同版本的ndk
##2、使用llvm-libc++类型的toolchain编译自己的代码

 我们使用cmake>=3.7.2版本和上一步生成的~/my-android-toolchain
 就可以很简单的编译我们自己项目的android版本

    cmake_minimum_required(VERSION 3.7)
    project(webrtc_test)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fpic")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fpic -std=c++11")
    include_directories(${PROJECT_SOURCE_DIR} "./include/")
    FILE(GLOB SOURCE
    "./*.cc")
    add_library(webrtc_test ${SOURCE})
    
使用如下命令来运行cmake:

    cmake -DCMAKE_SYSTEM_NAME=Android \
    -DCMAKE_ANDROID_STANDALONE_TOOLCHAIN=~/my-android-toolchain \
    -DCMAKE_BUILD_TYPE=Debug .. 
    cmake --build .
这样就完成了android sdk的编译，怎么样很简单吧？

注意这里一定要是用cmake 3.7.2的版本，因为低版本不支持DCMAKE_ANDROID_STANDALONE_TOOLCHAIN参数
目前测试cmake 3.8无法编译通过，所以推荐使用cmake 3.7.2

有的朋友以前可能会使用一个开源项目叫做android-cmake来编译android
里面有一个toolchain配置文件android.toolchain.cmake
但是目前这个文件只支持ndk版本小于等于r10d的版本
在高版本ndk中由于ndk目录结构发生改变，无法再使用android.toolchain.cmake文件了
所以建议大家以后放弃使用android.toolchain.cmake文件，而直接使用系统安装的cmake+standalone-toolchain即可。

## 3、android链接小技巧
android在链接时会依赖库的链接顺序，如果你觉得这样很难搞清楚先后顺序，可以试一下把所有静态库.a合并为一个大的整体库liball.a，在同一个库中的.o文件是没有顺序问题的。

*（本文基于webrtc branch55编写,在macos 10.12上测试）*

关键词：webrtc android 编译 webrtc链接