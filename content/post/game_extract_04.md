---
title: "[JNI查错] 口袋侦探闪退修复（三） - 分辨率缩放与触摸"
date: 2026-06-05T17:15:31+08:00
categories: ["逆向"]
tags: ["安卓", "JNI", "口袋侦探", "逆向"]
draft: false
---
**本章针对口袋侦探第一部在1080P设备上，画面过小只有544px的问题。** \
**希望本文的view欺骗思路，能够给cocos2dx同好修复一些老游戏的分辨率提供参考。**
## 核心观点
第一部的引擎太老，是cocos2dx-1.0.1的样子，加之下面的历史因素。
1. 厂家没在AppDelegate考虑缩放。 
2. setDesignResolution函数和缩放逻辑，是cocos2dx-1.1后期引入的。

## 可能的切入点
1. hook引擎调setFrameSize/setContentFactor/enableRetinaDisplay？\
&nbsp;&nbsp;结论：太麻烦了，还要重置视点和投影矩阵
2. 在framework侧，修改DPI？\
&nbsp;&nbsp;结论：这没用，又不是安卓原生布局。
3. 欺骗view控件，渲染侧，锁定720P的分辨率？\
&nbsp;&nbsp;结论：可行，推荐此法配合触摸点修复。
4. 修改手机分辨率wm size之类？\
&nbsp;&nbsp;结论：可行，但别失手搞出“赛博灯泡”。

## 修复详述
思路说来很有趣：安卓的视图基本上都是view和surface，native侧游戏总归在view下渲染。\
我们天天拿4k/2K的新手机，看720P视频也没见真就那么小吧。\
其实说不定只要改第二处，不过没有验证过。（fixedSize应该会传染给render?）\
（以下为代码等效，具体修改用smali或dex/jar联合覆盖）
1. so侧需要欺骗nativeInit，伪造尺寸上报
```java
// 直接跟setScreenWidthAndHeight，找到这里
public class Cocos2dxGLSurfaceView extends GLSurfaceView {
    @Override // android.view.View
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {

        // 开始修改
        w = 1280;
        h = 720;
        // 结束

        this.mRenderer.setScreenWidthAndHeight(w, h);
    }
}
```
2. VIEW控件一定存在缩放策略，伪造
```java
public class Cocos2dxGLSurfaceView extends GLSurfaceView {
    // ........
    protected void initView() {
        // ........

        // 开始追加VIEW侧，定尺寸伪造
        getHolder().setFixedSize(1280,720);
        // 结束

        this.mRenderer = new Cocos2dxRenderer();
        setFocusableInTouchMode(true);
        setRenderer(this.mRenderer);
        // ........
    }
    // ........
}
```

## 触摸修复
经过720P分辨率压制后，游戏画面在1080P设备会缩到左上角；\
再经由setFixedSize把egl缓冲区对齐到720P后，系统插值缩放铺满全屏。\
但是触摸还躲在老地方呢，无非是把1920x1080的大坐标报点，比例变成1280x720的逻辑小坐标。\
（伪代码如下/实际smali）
```java
public class Cocos2dxGLSurfaceView extends GLSurfaceView {
    @Override // android.view.View
    public boolean onTouchEvent(MotionEvent event) {
        int pointerNumber = event.getPointerCount();
        final int[] ids = new int[pointerNumber];
        final float[] xs = new float[pointerNumber];
        final float[] ys = new float[pointerNumber];

        // 片段1：在循环前算出缩放比例
        float scaleX = 1280.0f / (float) this.getWidth();
        float scaleY = 720.0f / (float) this.getHeight();
        // 片段1：结束

        for (int i = 0; i < pointerNumber; i++) {
            ids[i] = event.getPointerId(i);
            xs[i] = event.getX(i);
            ys[i] = event.getY(i);

            // 片段2：乘上zoom in的缩小比例映射
            xs[i] *= scaleX;
            ys[i] *= scaleY;
            // 片段2：结束
        }
        // ......
    }
}
```





## 后记
闪退、分辨率和伴生的触摸bug都修好了，童年已经补完，但谁也回不去过往了。\
ʅ（´◔౪◔）ʃ \
说起来关于闪退，之前怀疑过cocos2d引擎问题，但想想看一个时代的“**捕鱼达人**”不是还能玩吗？\
主线上对JNI的严格度还是很高的，最终定位到是厂家的sqliteJNI逻辑有问题。
