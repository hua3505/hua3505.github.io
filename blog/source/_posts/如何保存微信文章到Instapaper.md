---
title: 如何保存微信文章到Instapaper
date: 2018-01-08 17:54:52
tags:
- Android
---
## Instapaper实现稍后阅读
　　当在网上看到一篇好的文章时，而又没时间去读完它时，该怎么办呢。以前我会这样做：保持这个标签页不关闭；或者添加到浏览器的书签里；或者通过Evernote来记录文章的链接。但是常常不小心就把保留的标签页关了，而收藏到浏览器或者Evernote的文章也常常不记得回头去看。为了解决这个问题，我用了Instapaper。
![Instapaper移动端](http://upload-images.jianshu.io/upload_images/196189-1ef86684bc2e7323.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/465 "Instapaper移动端")
　　Instapaper的界面非常简洁，而且纯粹，非常好的聚焦在阅读文章上。这一点比Pocket要好。PC网页端，看到好的文章时，点一下浏览器插件或者Instapaper的书签就直接保存了。Android端，通过系统的分享选择Instapaper就可以保存文章。

从我切身体会，谈一下使用Instapaper的好处：
1. 随时保存，闲时阅读。
1. 统一阅读环境。Instapaper的简洁界面和提供的划线、记笔记，还有推到Evernote等功能非常棒。
1. 保存文章阅读进度，阅读长文章时特别有用。
1. 归档零散阅读的文章，方便以后查找。

## 一键保存微信文章到Instapaper
　　在使用Instapaper过程中发现一个问题：微信文章怎么保存到Instapaper。在网上搜索，大部分网友都是先用浏览器打开，然后用浏览器的分享去添加到Instapaper。尝试了一下，miui自带的浏览器的分享功能比较特殊，不能分享到Instapaper。而且就算可以，这个路径也有些太长了。身为一个Android程序猿，那必须想办法搞定这个问题。既然可以用浏览器打开，那干脆开发一个“浏览器”，用这个特殊的“浏览器”打开的时候就直接分享到Instapaper，这样就可以一步到位了。效果如下。

![保存到Instapaper](http://upload-images.jianshu.io/upload_images/196189-8b848822980ab5ac.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/465 "保存到Instapaper")
![保存成功](http://upload-images.jianshu.io/upload_images/196189-80aad7f4f5724f6e.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/465 "保存到Instapaper")
需要注意：
1. 如果点了用浏览器打开，却没弹出选择浏览器的界面的话，可能是以前选过“总是”。只能先清除一下微信数据（一定要谨慎，微信聊天记录没有在云端保存，最好先自己备份）。
1. 如果弹出了选择浏览器的界面，最好也不要选“总是”，因为以后可能会有一些场景真的需要用浏览器打开。

**下载地址：**
我已经把应用发布到应用宝上。通过微信扫描二维码就可以下载。也可以在应用宝里面搜索[“Instapaper微信保存插件”](http://android.myapp.com/myapp/detail.htm?apkName=com.gmail.huashadow.savetoinstapaper)。
![微信扫描下载](http://upload-images.jianshu.io/upload_images/196189-503ec5c4b37ea321.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "微信扫描下载")
安装完成之后不会有桌面图标，直接在微信里面使用就可以了。
欢迎大家在应用宝里给我评分，或者在文章底下给我留言，谢谢大家支持。

## 技术实现
[源码已通过github共享](https://github.com/hua3505/SaveToInstapaper)
**让我们的工具成为一个“浏览器”**
```xml
<activity
    android:name="com.gmail.huashadow.savetoinstapaper.ShareActivity"
    android:theme="@android:style/Theme.Translucent.NoTitleBar"
    android:icon="@mipmap/instapaper"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
​
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
​
        <data android:scheme="http" />
        <data android:scheme="https" />
    </intent-filter>
</activity>
```
**保存到Instapaper**
```java
private void saveToInstapaper() {
   Intent intent = getIntent();
   if (intent.getAction().equals(Intent.ACTION_VIEW)) {
       Log.d(TAG, intent.getData().toString());
       Intent saveToInstapaperIntent = new Intent(Intent.ACTION_SEND);
       saveToInstapaperIntent.setPackage("com.instapaper.android");
       saveToInstapaperIntent.setType("text/plain");
       saveToInstapaperIntent.putExtra(Intent.EXTRA_TEXT, intent.getData().toString());
       startActivity(saveToInstapaperIntent);
   }
   finish();
}
```
