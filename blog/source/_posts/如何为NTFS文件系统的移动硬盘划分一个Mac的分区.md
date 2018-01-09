---
title: 如何为NTFS文件系统的移动硬盘划分一个Mac的分区
date: 2018-01-03 14:14:16
tags:
---
### 1. 背景
事情是这样的，我准备编译android源码，但是我的MacPro只有256G的空间，而源码编译大概要100G左右的空间。于是，我从箱底翻出500G的硬盘。只有一个分区，NTFS文件系统的，编译源码需要Mac OS扩展（区分大小写，日志式）。但里面还有300多G的资料、软件、游戏之类的，直接格式化肯定不行，备份又麻烦。所以我准备在移动硬盘上划分一个新的分区，格式化成mac需要的格式。<!-- more -->

### 2. 压缩原有分区并创建Mac OS 扩展分区
##### 使用的工具
- Windows自带的磁盘管理工具
- Mac自带的磁盘管理工具

##### 具体步骤
1. 利用Windows自带的磁盘管理工具，压缩移动硬盘上原有的分区。据说编译源码100G的空间够了，于是我压缩出了100G左右的空间。
1. 利用Windows自带的磁盘管理工具，创建新的分区（使用压缩出的空间）。这里可能因为我的移动硬盘原有的分区是NTFS的，所以无法在Mac上对移动硬盘进行分区，只能先在Windows上分区。
1. 利用Mac自带的磁盘管理工具，格式化新建的分区。
![分区完成后](http://upload-images.jianshu.io/upload_images/196189-39457ca875c263d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "分区完成后")

### 3. 扩展MacOSX分区的大小
分区完成之后，下载Android源码进行编译，艰难绕过各种坑之后，在编译到80%时失败了，空间不足！！！于是只能想办法再给MacOSX分区扩展一点空间。
##### 工具
- Windows自带的磁盘管理工具
- iPartition（Mac上的一个第三方磁盘分区工具）

##### 具体步骤
1. 利用Windows自带的磁盘管理工具，压缩NTFS分区的空间。空间挤挤总是有的，又压缩出50G空间。
1. 利用iPartition工具，把压缩出的50G空间合并到MacOSX分区中。这一步尝试了Windows的磁盘管理工具、Mac的磁盘管理工具还有Windows上的分区助手都没法搞。最后用iPartition轻松搞定。
![空间扩展完成后](http://upload-images.jianshu.io/upload_images/196189-3729bcfa549834ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "空间扩展完成后")
