---
layout:     post
title:      "Real-Time Rendering, Graphics Hardware读书笔记"
subtitle:   "Graphics Hardware"
date:       2016-04-28 13:00:00
author:     "Candycat"
header-img: "img/in-post/graphics-hardware-bg.jpg"
tags:
    - Graphics Hardware
    - 读书笔记
---

## Chapter 18 - Graphics Hardware

尽管图形硬件变化很快，但还是有一些通用的概念架构设计。

## 18.1 缓存（Buffers and Buffering）

帧缓存的内存可能存在于和CPU同样的内存空间中（某个专用的帧缓存内存空间），也可能存在于**显存（video memory）***中。显存中包含了所有的GPU数据，但不能直接被CPU访问。颜色缓存（color buffer）是帧缓存（frame buffer）的一部分。它通过某种方式和一个**显示控制器（video controllder）**相连，显示控制器又和某个显示器相连。如下图所示。显示控制器通常包含了一个数字模拟转换器（digital-to-analog converter，DAC），数字模拟转换器负责把数字像素值转换成一个模拟信号进行显示。由于DAC需要在每一帧时对所有像素都进行转换，因此系统必须拥有很高的带宽（bandwidth）。基于CRT的设备是模拟设备（analog devices），因此它需要接受模拟输入（意味着之前需要把数字信息转换成模拟信号再传递给它进行显示）。而基于LCD的显示设备是数字设备（digital devices），但它们通常既可以接受模拟输入也可以接受数字输入。我们的个人电脑使用的是数字视频接口（digital visual interface，DVI），也被称为显示端口接口（Display Port digital interfaces），而诸如游戏系统和电视机这样的电子设备则通常使用的是高清晰度多媒体接口（high-definition multimedia interface，HDMI）。

![img](/img/in-post/simple-display-system.png)
图示：一个简单的显示系统：显示控制器负责扫描颜色缓存，得到每个像素的颜色值后将其用于控制输出设备的显示强度。

阴极射线管（CRT）显示器刷新图像的速率通常在60到120Hz/s。显示控制器的任务就是扫描整个颜色缓存，一条扫描线接着一条扫描线。显示控制器是和显示器的电子束（beam）同步工作的，也就是说它们的扫描速率是一致的（可以这样理解，显示器的电子束扫描到哪里，就会把显示屏的该区域点亮，与此同时显示控制器也按照同样的速率读取颜色缓存，并将其作为输入提供给显示器的电子束来控制显示器的显示亮度，因此两者扫描速率是同步的）。需要注意的是，电子束的移动轨迹通常是从左到右、从上到下的。因此，当电子束扫描完一行需要重新从右边移到左边扫描下一行时，这个从右移到左的过程是不会对屏幕图像产生任何影响的。这被称为**水平回扫（horizontal retrace）**。与之相关的还有**水平刷新率（horizontal refresh rate）**，这个值就是指完成一次完整的左-右-左过程所需要的时间。而**垂直回扫（vertical retrace）**就是指电子束从右下移动到左上开始扫描下一帧的过程，同样，**垂直刷新率（vertical refresh rate）**就是每秒完成这个过程的次数。当它小于72Hz时，大多数用户就会察觉到画面跳帧了。这个过程如下图所示。

![img](/img/in-post/monitor-retraces.png)
图示：显示器的水平和垂直回扫。这里显示的颜色缓存有5行。扫描线开始于左上角，然后每次会扫描一行。当扫描到一行的结尾时，它需要移动到下一行的开头。这个过程就是水平回扫。当扫描到最后的右下角时，本次扫描就结束了，就需要再移动到左上角来进行下一帧的扫描。这个移动过程被称为垂直回扫。

液晶显示器（ liquid crystal display，LCD）通常的刷新率为60Hz。而CRT需要更高的刷新率（60到120Hz）是因为当电子束移开后显示器上的磷光剂就开始逐渐变暗了（因此如果不快点的话之前扫过的就都变得很暗了）。而LCD可以持续传输光，因此不需要很高的刷新率。而且LCD也不需要回扫时间。但LCD同样有垂直同步的概念，这里的刷新率会受当前生成哪些帧的影响（参考2.2节）。

与该话题相关的是隔行扫描（interlacing）。计算机显示器通常不是隔行扫描的，而是逐行扫描（progressive scan）的。而电视机则是隔行扫描的，也就是说，在一次垂直刷新过程中，奇数行会被绘制，然后在下一个刷新过程中，偶数行会被绘制。