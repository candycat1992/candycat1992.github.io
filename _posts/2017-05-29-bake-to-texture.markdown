---
layout:     post
title:      "Bake Shading to Texture踩坑记录"
subtitle:   "踩坑记录"
date:       2017-05-29 00:00:01
author:     "Candycat"
header-img: "img/in-post/default-bg.jpg"
tags:
    - 工作记录
---

## 写在前面

最近需要完成这样一个需求，在此记录下踩的坑。这种需求还是比较常见的，比如[皮肤渲染](https://zhuanlan.zhihu.com/p/27014447)里会把diffusion profile存储在图像空间里，进行处理后再贴回到模型表面。

## 流程

首先大概讲一下流程吧。

1. 把shading的结果根据模型的uv烘焙到一张固定大小的texture上（假定摄像机背景色为黑色），我们称之为baked texture
2. 在实际渲染的时候再根据模型uv贴回去，理论上来说可以得到和直接渲染一样的渲染效果

下面这张就是把模型简单的漫反射结果根据uv烘焙到纹理上得到的baked texture，它的layout取决于uv展开时的layout。

![baked-texture.png](http://candycat1992.github.io/img/in-post/2017-05-29-bake-to-texture/baked-texture.png)

## 接缝问题

然而，当把这张图贴回模型的时候，我们可以发现在明暗变化比较明显的地方（通常是烘焙贴图分辨率低于屏幕像素密度）或者在uv接缝处会有缝隙。

![seam-texture0.png](http://candycat1992.github.io/img/in-post/2017-05-29-bake-to-texture/seam-texture0.png)

## 原因

上面那张图还是经过双重滤波后的样子。如果我们用point filter去采样这张baked texture来贴到模型上，就可以发现那些明显的异常像素点：

![seam-texture1.png](http://candycat1992.github.io/img/in-post/2017-05-29-bake-to-texture/seam-texture1.png)

这些黑色的点是因为采样到了背景像素（在前面我们把摄像机的背景色设成了黑色）。可以分析原因如下。这些位置处的顶点会存在UV Split的情况，虽然顶点位置一样，但在物理上它们会对应多个具有不同uv值的顶点。

![seam-texture2.png](http://candycat1992.github.io/img/in-post/2017-05-29-bake-to-texture/seam-texture2.png)

对于这样接缝边相邻的两个三角形，由于这条边的两个顶点在两个三角形内完全是不同的uv坐标，所以在光栅化的时候它们实际上光栅化到了baked texture上完全不相邻的两个位置。当我们重新要把这张baked texture贴上来的时候，采样到接缝处时很有可能就采样到旁边的背景像素。如果开始了纹理filter和mipmap，则会因为uv导数不连续，同样导致无法得到正确的结果。

实际上除了这些黑色像素，你还可以发现出现了棋盘状的一些效果。我们来尝试还原这些黑色像素和棋盘状效果的产生过程。我们把最后输出时候的像素标记为$P_s$，烘焙时候的像素称为$P_b$。那么过程如下：

1. 在第一步烘焙的时候，在vertex shader阶段我们根据模型uv输出顶点的屏幕位置，然后经过处理后进入光栅化。由于接缝处uv不连续，因此实际会光栅化到baked texture的完全不相邻的位置上去。同时，由于baked texture的分辨率为512x512，因此光栅化时插值的uv精度（即更新baked texture的精度）也会是1/512。然后我们进行了正常的光照计算，把结果存到了这些光栅化像素$P_b$中。
2. 在把baked texture贴回去的时候，经过屏幕空间光栅化后生成了$P_s$，这次$P_s$会按照模型uv坐标去查找baked texture。由于纹理分辨率问题，多个$P_s$会采样到同一个$P_b$，所以会出现图中棋盘状的结果，这是因为每一个格子内的$P_s$对应了同一个$P_b$。而黑色像素则是因为，这个精度位置的$P_b$压根在烘焙的时候压根就没有被更新到，所以还会保留着背景颜色。

既然如此，要想消灭这些背景像素，理所当然想要的一个方法就是提高baked texture的分辨率。提高分辨率这样的确可以减少黑色像素，但还是无法完全消灭。下面是baked texture为4096x4096，屏幕大小为800x800时候的情况：

![seam-texture3.png](http://candycat1992.github.io/img/in-post/2017-05-29-bake-to-texture/seam-texture3.png)

这主要是因为烘焙时候的精度和uv展开的layout也有关系，这造成总会有一些屏幕空间的像素无法在烘焙的时候得到更新，导致仍然保留的背景色。一个解决方法是把baked texture向背景部分bleed几个像素，使得这些接缝处的像素可以向外扩展一些。这样的确可以在baked texture分辨率较高的情况下清除掉这些背景像素：

![seam-texture4.png](http://candycat1992.github.io/img/in-post/2017-05-29-bake-to-texture/seam-texture4.png)

但是，在baked texture分辨率较低、且接缝处明暗变化较大时，还是会出现肉眼可见的接缝。下面是baked texture为512x512、屏幕分辨率为800x800，在bleed前后的对比图（bleed时取背景像素点为中心的3x3方格内所有非背景像素点的平均值）：

![seam-texture5.png](http://candycat1992.github.io/img/in-post/2017-05-29-bake-to-texture/seam-texture5.png) ![seam-texture6.png](http://candycat1992.github.io/img/in-post/2017-05-29-bake-to-texture/seam-texture6.png)

可以从右图明显地看到不连续和残留的黑色背景像素点。背景残留还是因为精度差别较大导致bleed的像素个数不够，当然你可以通过增加bleed的宽度，但这样无疑会增加采样点，baked texture分辨率越低，需要增加的宽度就越多。而不连续一方面是因为此时棋盘格子（由于uv layout以及采样到了同一个$P_b$）比较明显，一方面是bleed的时候我们只把当前这一边的像素扩展出去，而没有真的去混合uv接缝处实际的另一边的像素值。

## 总结

理论上，要从根本上解决接缝处的问题，我们应该要知道接缝处真正的旁边像素是什么，即修改它的偏导，这样就可以真的去混合正确的相邻颜色得到平滑的结果，但这难做到。或者是真的去“膨胀边界”，这和建模软件里展UV道理相同，可以想象为什么普通的贴图没有接缝问题，这是因为建模软件在展开的时候可以选择在边界处多渲染一个宽度，这个宽度内的像素不是像我们上面那样简单的复制或取当前的平均值，而是找到真正相邻位置的信息渲染到这个边界处，从而在采样的时候就可以得到正确的混合结果，但这个渲染的过程在实时里负担比较重。

实际上，这个问题和烘焙lightmap时候出现的light/shadow bleeding问题很像。大部分解决方法就是向背景部分膨胀，也就是我们上面提到的bleed方法。或者是提高分辨率。

Nvidia在[2007 Advanced Skin](http://developer.download.nvidia.com/presentations/2007/gdc/Advanced_Skin.pdf)里也提到了一些方法：

* Change clear color（鸡肋）
* Object / no-object alpha map（应该就是类似我们这里的bleed）
* Multiple UV sets

额，最后一个不知道是什么东西。

总之，到了这里，对于我的需求来说，我决定就此放弃这个方法了……