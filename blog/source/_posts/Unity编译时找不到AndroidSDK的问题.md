---
title: Unity编译时找不到AndroidSDK的问题 | Unable to list target platforms
date: 2018-01-08 18:12:29
tags:
- Unity
- Android
---
## 现象
在用 Unity 编译 Android 平台的应用时，遇到 Unable to list target platforms 的问题。

![错误提示](http://upload-images.jianshu.io/upload_images/196189-9153d06c787e7c2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

详细错误描述如下：
> Error:Invalid command android
UnityEditor.BuildPlayerWindow:BuildPlayerAndRun()

> CommandInvokationFailure: Unable to list target platforms. Please make sure the android sdk path is correct. See the Console for more details. 
C:\Program Files\Java\jdk1.8.0_91\bin\java.exe -Xmx2048M -Dcom.android.sdkmanager.toolsdir="D:/Android/sdk\tools" -Dfile.encoding=UTF8 -jar "D:\Program Files\Unity\Editor\Data\PlaybackEngines\AndroidPlayer/Tools\sdktools.jar" -
> stderr[
Error:Invalid command android
]
stdout[
> ]
UnityEditor.Android.Command.Run (System.Diagnostics.ProcessStartInfo psi, UnityEditor.Android.WaitingForProcessToExit waitingForProcessToExit, System.String errorMsg)
UnityEditor.Android.AndroidSDKTools.RunCommandInternal (System.String javaExe, System.String sdkToolsDir, System.String[] sdkToolCommand, Int32 memoryMB, System.String workingdir, UnityEditor.Android.WaitingForProcessToExit waitingForProcessToExit, System.String errorMsg)
UnityEditor.Android.AndroidSDKTools.RunCommandSafe (System.String javaExe, System.String sdkToolsDir, System.String[] sdkToolCommand, Int32 memoryMB, System.String workingdir, UnityEditor.Android.WaitingForProcessToExit waitingForProcessToExit, System.String errorMsg)
UnityEditor.BuildPlayerWindow:BuildPlayerAndRun()

## 原因
### 原因简单描述
Unity 在编译时会调用 Android SDK tools 中的 android 命令，而在新版本的 Android SDK tools 中，android这个命令已经废弃了，导致 Unity 无法正常编译。我的 Android SDK tools 版本是 25.3.1 。
### 找到问题原因的过程
经过再三确认，我配置的 Android SDK 是没问题的。
![SDK设置](http://upload-images.jianshu.io/upload_images/196189-5b2c7c2750b37566.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "SDK设置")
后来我注意到错误描述中有提到“无效的命令 android ”，所以我尝试直接调用 android 这个命令，看是不是有问题。
> Error:Invalid command android

android 命令是 AndroidSDK 中 tools 目录下的 android.bat 。直接调用，发现这个命令已经废弃了。
> The "android" command is no longer available.
For manual SDK and AVD management, please use Android Studio.
For command-line tools, use
tools\bin\sdkmanager.bat and tools\bin\avdmanager.bat

## 解决方案
知道了原因，问题就好解决了。
1. 从官网下载一个旧版本的 Android SDK tools 。[tools_r25.2.3-windows.zip](https://dl.google.com/android/repository/tools_r25.2.3-windows.zip)。
1. 把原来 SDK 目录下的 tools 备份一下。我是把它重命名成 tools-25.3.1 。
1. 把下载好的旧版本的 tools 解压到 SDK 目录下。
1. 再在 Unity 中重新编译，问题已经解决了。

对比了一下两个版本的 tools，差别真的很大，少了很多东西。比如，做.9图的 draw9patch 就不知道去哪里了。

![tools 25.2.3 vs 25.3.1](http://upload-images.jianshu.io/upload_images/196189-287b910ccfd1f58a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "tools 25.2.3 vs 25.3.1")
