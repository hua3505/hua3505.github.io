---
title: Android控件设置宽高比
date: 2018-02-13 17:45:18
tags: 
- Android
- Data Binding
---
## 0. 困扰很久的问题

Android控件的宽和高保持比例，这是从我接触Android以来，一直不断会遇到的需求。以前，要么就是在代码里直接设置宽和高，要么就是自定义控件。网上也有开源的自定义ViewGroup，可以让其子View比较方便的设置宽和高的比例。但这些实现方式，还是比较麻烦，也不够直观。直到有了DataBinding，我们可以很方便地给控件加上自定义的属性，也就可以很方便的在布局文件中设置控件的宽高比了。<!-- more -->

## 1. 如何实现 

通过BinderAdapter为所有View绑定下面的方法，当设置widthHeightRatio属性时，会调用下面这个方法。这个有点AOP的意思，我们针对所有的View做了处理。

```java
public class DataBindingAdapters {
    // 根据View的高度和宽高比，设置高度
    @BindingAdapter("widthHeightRatio")
    public static void setWidthHeightRatio(final View view, final float ratio) {
        view.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                int height = view.getHeight();
                if (height > 0) {
                    view.getLayoutParams().width = (int) (height * ratio);
                    view.invalidate();
                    view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
                }
            }
        });
    }
}
```

我们在获取到控件的高度后，根据比例计算宽度，然后设置给控件。这里注册了OnGlobalLayoutListener，是因为控件的高度有可能还没计算完成。在获取到高度之后，移除监听，避免多余的调用。

```xml
<ImageView
   android:layout_width="120dp"
   android:layout_height="match_parent"
   app:widthHeightRatio="@{1}"/>
```

然后我们就可以在布局文件中直接设置宽高比了。这个布局文件必须使用DataBinding，也就是最外层要用layout标签。属性值必须加上@{}，不然是按普通属性处理的，不会调用我们的方法，编译时会因为找不到属性报错。当然，这个属性只能根据高度计算宽度，如果要根据宽度计算高度，可以用同样的方式再加一个属性。

## 2. 原理简析

其实在编译后的layout文件中是没有我们加的属性的（编译后的layout文件在build/intermediates/data-binding-layout-out下面可以看到）。真正设置这个属性，还是在Java代码中直接调用了我们绑定的方法。在DataBinding自动生成的Binding类中，可以发现有类似下面这样的调用。

```java
DataBindingAdapters.setWidthHeightRatio(this.mboundView0, cardWidthHeightRatio);
```

这里只是做一个简单的解释，至于在什么时机会触发这行代码，敬请期待我后续的文章。

## 3. BinderAdapter的其他妙用

### ImageView自动加载网络图片

```java
@BindingAdapter({"android:src", "error"})
public static void setImageUrl(ImageView view, String url, @DrawableRes int errorImage) {
    if (!TextUtils.isEmpty(url)) {
        // 封装好的图片加载工具
        ImageManager.from(view.getContext()).displayImage(view, url, errorImage);
    } else {
        view.setImageResource(errorImage);
    }
}
```

直接在布局文件中设置要加载的图片的url，和加载失败时显示的默认图。
