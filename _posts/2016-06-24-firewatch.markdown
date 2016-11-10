---
layout:     post
title:      "Firewatch中的一些技术"
subtitle:   "Game Art Tricks"
date:       2016-06-24 12:00:00
author:     "Candycat"
header-img: "img/in-post/firewatch.jpg"
tags:
    - Game Art Tricks
---

视频：[GDC 2016](http://www.gdcvault.com/play/1023191/Making-the-World-of)，[GDC 2015](http://www.gdcvault.com/play/1022295/The-Art-of)

开发公司是Campo Santo，只有十几个人（但大多工作多年，很有经验），感觉更像是Indie Game。开发时间两年。

## 艺术风格

### Layers of Colors

![屏幕快照 2016-06-24 21.44.20.png-59.4kB][1]

天空盒子是procedural的，着色使用了Unity的插件Skybox，来实现image-based lighting。

![屏幕快照 2016-06-24 21.45.30.png-235.9kB][2]

除了光照有layers以外，fog也是一个非常重要的因素。使用了Stylistic fog，一种additive blend post process，使用了渐变纹理。

![屏幕快照 2016-06-24 21.51.51.png-923.4kB][3]

![屏幕快照 2016-06-24 21.52.02.png-849.7kB][4]

### 使用的插件

![屏幕快照 2016-06-24 21.47.48.png-132.5kB][5]

SECTR：用于场景构建。

## World Streaming

Open world需要适时load/unload场景资源。

![屏幕快照 2016-06-24 21.31.25.png-218.2kB][6]

使用unload barrier（绿色）和load barrier（粉色），判断玩家是否越过了这些barrier，来unload或者load适当的资源。

![屏幕快照 2016-06-24 21.31.44.png-560kB][7]

场景中设置的两种barrier，人工量挺大的，要找好点。

这种还能用来加载不同load不同的burned version（70天之前是非烧毁的场景，70天之后就是烧毁的场景）。

## 树

FireWatch里有很多树，Campo Santo当时用的Unity版本是Unity 4.5，而SpeedTree是5.0才引进的，因此他们选择手动建模（4.5版本的Tree Creato just sucks...）。

![屏幕快照 2016-06-24 20.46.54.png-366.4kB][8]

游戏里所有的树。

![屏幕快照 2016-06-24 20.49.12.png-505.5kB][9]

左图：Maya中的效果。中间：Unity中的渲染效果。右侧：使用的纹理。

注意，树的底部细节更丰富，包含了更多真正的geometry的结构，这是因为主角其实相对于树涞水很矮，因此下部的细节应该更丰富。

### LOD技术

![屏幕快照 2016-06-24 20.48.38.png-416.4kB][10]

手工制作各种树的LOD，共5个。没有使用LOD cross fading，因为可能会在fading时造成造成更多的draw call。

### Alpha Test

![屏幕快照 2016-06-24 20.54.29.png-505.3kB][11]

离摄像机越远，alpha cutoff的值就越小，让树看起来更膨。很简单的方法来得到想要的效果。

## Custom Tree Wind Respond

![屏幕快照 2016-06-24 20.56.43.png-285.5kB][12]

使用了vertex color + vertex animation，方法简单，但效果非常好。

## Terrain

只用了Unity自带的Terrain，使用简单。但是，有一些问题，默认情况下，Unity的terrain textures是top-down projected的，有很多uv streches。

![屏幕快照 2016-06-24 21.01.16.png-415.9kB][13]

他们避免麻烦的方法，使用石头网格来覆盖，但人工量很大。

另一个是grass的问题，Unity内置的grass很好使用，可以调整大小、颜色等等，但是性能有问题。小心Detail Map的Resolution。

还有就是Unity Terrain的tessellation问题，他们关闭了terrain的shadow casting，然后使用自己自定义的shadow casting mesh。

  [1]: http://static.zybuluo.com/candycat/8gve5teno4tt9sr4kmhzmrz6/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2021.44.20.png
  [2]: http://static.zybuluo.com/candycat/f4745med0z0mcwnxf3a849z7/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2021.45.30.png
  [3]: http://static.zybuluo.com/candycat/ya638s6g29mbyhitqyssrp4c/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2021.51.51.png
  [4]: http://static.zybuluo.com/candycat/117jbjb3tbw9qihb9xx1k2jg/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2021.52.02.png
  [5]: http://static.zybuluo.com/candycat/y2rt5dw7iet3cf3ubi1v4kr4/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2021.47.48.png
  [6]: http://static.zybuluo.com/candycat/k4k9evy9ni3yn4f3eieo55tl/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2021.31.25.png
  [7]: http://static.zybuluo.com/candycat/5xalhltkxr6f69t47wux5o8s/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2021.31.44.png
  [8]: http://static.zybuluo.com/candycat/pkcdhe19zk9zkzjvlw5c63mh/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2020.46.54.png
  [9]: http://static.zybuluo.com/candycat/ks2z9e3bkr66sxgpbsn9ant2/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2020.49.12.png
  [10]: http://static.zybuluo.com/candycat/fjnmgss09lrn7c50futkxfgv/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2020.48.38.png
  [11]: http://static.zybuluo.com/candycat/iqzltuego4zglfv6ro1muzv0/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2020.54.29.png
  [12]: http://static.zybuluo.com/candycat/ezouupsmmztktmzes7o9jay7/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2020.56.43.png
  [13]: http://static.zybuluo.com/candycat/k6zxxhlt50kifwmrtynr5wck/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-06-24%2021.01.16.png