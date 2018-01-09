---
title: 在简书上用Markdown画流程图
date: 2018-01-08 18:02:48
tags:
- Markdown
- 简书
---
作为一个程序员，经常需要画流程图。而用Markdown画流程图，省心省力，后面需要调整的话也更方便。但是，目前简书上的Markdown还不支持流程图。怎么办呢？只能以迂为直，曲线救国了。

### 简单的说：
简书的Markdown本身不支持流程图，但我们可以找一个支持流程图的Markdown编辑器，画完之后，直接截图上传到简书上就可以了。但是有时候流程图太长，超出一个屏幕的高度，这时候需要用能滚动截图的工具来截。<!-- more -->

### 你将需要：
1. 一款支持流程图的Markdown编辑器。推荐Typora。
1. 一款截图工具。QQ自带的就可以。
1. 一款支持截长图的工具。Mac上可以用Snip。Windows上可以用FSCapture。

### 编辑流程图
第一步，当然是先写好我们的流程图了。这里我使用的工具是Typora。需要注意的是，Typora默认是不支持流程图的，需要在设置中开启这个功能。在设置中切换到Markdown那个tab，然后勾选上对图表（包含流程图）的支持。
![Typora勾选支持流程图](http://upload-images.jianshu.io/upload_images/196189-a333d53488c72560.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640 "Typora勾选支持流程图")

Markdown画流程图的语法可以看这篇文章：[Markdown笔记：如何画流程图](https://segmentfault.com/a/1190000006247465)。

先来画一个简单的流程图。
```flow
st=>start: 开始
rain?=>condition: 今天有雨吗？
takeAnUmbrella=>operation: 带伞
go=>operation: 出门
e=>end: 结束

st->rain?
rain?(yes)->takeAnUmbrella->go
rain?(no)->go->e
```

### 截图
下面到了收获成果的时候了，直接在Typora里面截图，然后把图片拖到简书的编辑器里就可以了。
**小贴士：**
- Typora也支持导出图片，不过如果你的流程图太宽的话，导出的图片不能包含整个流程图。
- 如果你的流程图确实非常宽，以致于在Typora里需要横向滚动才能显示完全的话，可以先导出成html，在浏览器里面打开，再进行截图。
![出门带伞的流程](http://upload-images.jianshu.io/upload_images/196189-9026e30aa8f37aed.png?imageMogr2/auto-orient/strip%7CimageView2/2/h/640 "出门带伞的流程")

### 截长图
如果流程比较复杂，流程图的长度超过一个屏幕的高度的话，就需要用到滚动截图的工具了。把Typora编辑好的流程图导出成html，然后用支持滚动截图的工具来截。
Mac下Snip可以完全滚动截图的操作，Snip在安装和使用的时候有几点要注意的地方，所以我简单介绍一下。
#### 1. 下载地址
[Snip官网](http://snip.qq.com)，只支持Mac。
#### 2. 安装
在安装时，如果直接双击打开，会提示“来自身份不明的开发者”，而无法打开。但这个Snip是腾讯做的东西，安全性应该是没有问题的，我们在这里用右键点击，然后选择打开，就可以安装了。
![双击安装会提示打不开](http://upload-images.jianshu.io/upload_images/196189-ccdb517532b13ece.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640 "双击安装会提示打不开")
#### 3. 设置
1. 首先需要在系统设置里面，找到安全性与隐私。在辅助功能里，勾选允许Snip控制您的电脑。我猜想是因为Snip在截图的时候需要滚动屏幕，所以需要这个权限。
1. 在Snip的偏好设置里，勾选启动滚动截屏。 
![Snip设置](http://upload-images.jianshu.io/upload_images/196189-aa8b6213f3778ce7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/960 "Snip设置")

#### 4. 使用
如果需要滚动截屏，在点击截屏后，直接单击，就会开始滚动截屏。这样截的图可能两边会有较多空白，可以用图片处理工具把多余的部分裁掉（Mac上可以直接用自带的图片预览工具进行裁剪）。
