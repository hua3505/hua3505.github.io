---
title: Unity开发中的常用操作和注意事项
date: 2018-01-08 17:03:37
tags:
- Unity
---
### 场景视角变换
平移：鼠标切换到手型，按下左键拖动
缩放：鼠标滚轮滚动；或者按下alt+鼠标右键，然后移动鼠标
围绕观察点旋转：按下alt+鼠标左键，然后移动鼠标
观察者旋转：按下鼠标右键，然后移动鼠标
聚焦到物体：选中物体后按F；或者在hierarchy中双击要聚焦的物体

### 编辑器布局切换
在编辑器上方菜单栏的Windows->Layouts可以切换不同的布局。有预设的一些布局，也可以自己修改布局然后保存。预设的里面我用的比较多的是*Default*和*2 by 3*。Default布局是中间上方有一个很大的场景窗口，可以通过标签页切换到Game窗口；2 by 3是左边分成上下两块，分别显示场景窗口和Game窗口。

### VisualStudio
首先要安装Visual Studio Tools for Unity。Unity在安装的时候就在VisualStudio中安装了这个插件，如果没有的话可以在VS的插件管理器（Tools->Extensions and Updates）中下载安装，或者去[msdn](https://visualstudiogallery.msdn.microsoft.com/site/search?f%5B0%5D.Type=RootCategory&f%5B0%5D.Value=tools&f%5B1%5D.Type=Tag&f%5B1%5D.Value=unity%203D%20script%20c%23%20debug%20breakpoint)上下载。

查看unity文档：Help->Unity API Reference，默认快捷键是 Ctrl + Alt +M,Ctrl + H。会在VS中打开Unity的在线文档，并查找当前选中的内容。
快速实现MonoBehaviour中的方法：鼠标右键->Implement MonoBehaviours(快捷键：Ctrl + Shift + M)。列出MonoBehaviour中所有可实现的方法，通过勾选添加。
查看MonoBehaviour中的方法：鼠标右键->Quick MonoBehaviours（快捷键：Ctrl+shift +Q）。查找MonoBehaviour中可实现的方法。
调试脚本：点击Attach to Unity，或者直接按F5连接到Unity上，在Unity中运行游戏，就可以开始断点调试了。

### 注意：不要在play模式下进行编辑
play模式下场景中的修改（场景中创建物体、修改物体属性等）都是临时的，退出play模式时就会还原。play模式下只适合用来做测试性的修改，而不应该在play模式下编辑。
问题来了，如何避免不小心在play模式下编辑的情况？
可以让Unity在play模式下显示不一样的颜色来提醒自己。在Preference的Colors中修改Playmode tint即可。效果见下图。

![修改Unity在Play模式下的颜色](http://upload-images.jianshu.io/upload_images/196189-715f1865280266cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "修改Unity在Play模式下的颜色")

### 版本管理：
参考：[在Unity项目中使用Git](http://www.cnblogs.com/CodeCabin/p/3960453.html)
1. 在 Edit->Project Settings->Editor->Version Control Mode 中选择 Visible Meta files。
1. 在 Edit->Project Settings->Editor->Asset Serialization Mode 中选择 Force Text。
1. 添加gitignore。

下面是Unity官方教程里面给出的gitignore

    =============== 
    Unity generated
    ===============
    Temp/
    Library/
    
    =====================================
    Visual Studio / MonoDevelop generated
    =====================================
    ExportedObj/
    obj/
    *.svd
    *.userprefs
    /*.csproj
    *.pidb
    *.suo
    /*.sln
    *.user
    *.unityproj
    *.booproj
    
    ============
    OS generated
    ============
    .DS_Store
    .DS_Store?
    ._*
    .Spotlight-V100
    .Trashes
    ehthumbs.db
    Thumbs.db

