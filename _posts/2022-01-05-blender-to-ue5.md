---
layout:     post
title:      "「闲谈」UE5真好玩"
subtitle:   ""
date:       2022-01-05 00:49:02
author:     "Candycat"
header-img: "img/in-post/default-bg.jpg"
tags:
    - 闲谈
---

## 前言

这两天生病休息，趁着有时间想要找点有意思的事情做。之前偶然看到Blender的[Art Gallery](https://cloud.blender.org/p/gallery/)，里面有很多艺术家开源的各种风格的Blender原工程文件，想着能不能在引擎里复现下看看能复现到什么程度，从里面我挑了自己比较喜欢的[Blender 2.81的Splash Screen](https://cloud.blender.org/p/gallery/5dd6d7044441651fa3decb56)项目。

一开始本来想在Unity HDRP里做做看，无奈搞了一会实在是劝退，跟UE相比Unity缺少很多查看Lighting的快捷方式，遂弃，还是乖乖滚回UE的怀抱。

## 结果

那就废话不多说直接UE5搞起。一顿操作下来初见成效，不得不说UE5真强啊啧啧啧。我们先来看下分别用Blender的Cycles和UE5实时渲染的对比：

**Blender Cycles渲染结果**：

![Snipaste_2022-01-05_23-22-03](http://candycat1992.github.io/img/in-post/2022-01-05-blender-to-ue5/Snipaste_2022-01-05_23-22-03.jpg)

**UE5实时渲染结果**：

![HighresScreenshot00007](http://candycat1992.github.io/img/in-post/2022-01-05-blender-to-ue5/HighresScreenshot00007.png)

（没头发的萌妹子秒变打工人，头发真的很重要）尽管UE5缺少某些Blender的渲染features，比如光源的软阴影、面光源计算与离线有差异等，但也十分接近了。

宇宙的尽头果然是UE5。
