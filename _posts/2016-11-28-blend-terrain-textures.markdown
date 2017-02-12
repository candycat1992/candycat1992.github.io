---
layout:     post
title:      "【Unity Shader】地形纹理合并"
subtitle:   "Unity Shader"
date:       2016-11-28 00:00:01
author:     "Candycat"
header-img: "img/in-post/2016-11-28-blend-terrain-textures/terrain_title.jpg"
tags:
    - Game Art Tricks
---

[题图链接](https://forum.unity3d.com/threads/terraincomposer-to-create-aaa-realistc-terrains-released.171928/)

# 写在前面

很长时间没更新博客，终于找到工作啦，相信会是一个很好的开始哇哈哈哈~~~

前一段时间有个朋友找我分析一个地形的shader，代码很长就主要看了下里面的纹理合并的部分。Unity目前常见的地形应该是T4M的做法，据说是（啊我自己没有用过…）只支持打包4张纹理，也就是说可以在地形上刷4种纹理，多了的话就不太方便了。而这个shader中的方法可以打包9张纹理，然后靠一张混合纹理来控制混合，感觉还挺巧妙的。

# 方法

这个shader主要会利用两张纹理。一张自然是包含了9种地形纹理的atlas纹理，就称为BlockMainTex吧：

![block_main](http://candycat1992.github.io/img/in-post/2016-11-28-blend-terrain-textures/block_main.png)

以及一张负责混合纹理的BlendTex：

![block_blend](http://candycat1992.github.io/img/in-post/2016-11-28-blend-terrain-textures/block_blend.png)

这张纹理是关键所在，它的RG通道存储了该位置处需要混合的**两种**地形纹理的**索引值**，它的B通道存储了这两种纹理的混合系数。下面是上图的RGB通道图：

![block_blend_rgb](http://candycat1992.github.io/img/in-post/2016-11-28-blend-terrain-textures/block_blend_rgb.png)

我们最终可以得到类似下面的效果：

![terrain_blend](http://candycat1992.github.io/img/in-post/2016-11-28-blend-terrain-textures/terrain_blend.png)

可以看出来，我们可以用一个draw call+两张纹理刷出至多9种不同的地形纹理。

## 地形纹理的索引

都可以看出来关键在于混合纹理BlendTex，它的RG通道存储了该位置处需要混合的两种地形纹理的索引值，即每个通道存储了一个索引值。实际上，由于BlockMainTex是按照九宫格来打包了9种纹理，所以这个索引是一个二维的向量（x，y），也就是说把这个二维（x，y）索引值打包进一个0~1的8 bits小数内（通道值的范围）。这主要是靠下面的公式：

$$f = \frac{x}{16} + \frac{y}{256}$$

其中，x和y分别表示在索引对应的行列值（我总是把上面的公式理解成把x编码进了前4个bits，把y编码进了后4个bits）。

上面是编码的过程，解码的相关公式就是：

$$x = floor(16f)\\y = floor(256f) - 16x$$

Shader代码对应：

```
float2 encodedIndices = tex2D(_BlendTex, i.uv).xy;

float2 twoVerticalIndices = floor((encodedIndices * 16.0));
float2 twoHorizontalIndices = (floor((encodedIndices * 256.0)) - (16.0 * twoVerticalIndices));

float4 decodedIndices;
decodedIndices.x = twoHorizontalIndices.x;
decodedIndices.y = twoVerticalIndices.x;
decodedIndices.z = twoHorizontalIndices.y;
decodedIndices.w = twoVerticalIndices.y;
decodedIndices = floor(decodedIndices/4)/4;				
```

decodedIndices就是0~3的整数索引值除以4的结果，即该种纹理在BlockMainTex中的起始值。拿图中樱花那个block举例，它对应的xy值是（0，8）（由于xy的范围是0~15，而图片索引范围是0~3，所以要乘以4），所以在BlendTex中的颜色就是8/256。

## 纹理采样

知道了两张地形纹理的索引，就该对它们进行采样了。

```
float2 worldScale = (worldPos.xz * _BlockScale);
float2 worldUv = 0.234375 * frac(worldScale) + 0.0078125; // 0.0078125 ~ 0.2421875, the range of a block

float2 uv0 = worldUv.xy + decodedIndices.xy;
float2 uv1 = worldUv.xy + decodedIndices.zw;
```

整个地形使用xz平面的世界坐标的小数部分作为采样坐标进行平铺。由于每个block其实只占了1/4的长宽值，所以要进行缩放。为了防止接缝处出现问题，还在两边稍微拉伸了下，即每边拉伸了0.0078125个单位（即1/128个单位）：

![terrain_blur](http://candycat1992.github.io/img/in-post/2016-11-28-blend-terrain-textures/terrain_blur.png)

## 处理接缝

如果直接使用上面的uv0和uv1对纹理采样，那么在地形接缝处会出现明显的问题：

![terrain_seam](http://candycat1992.github.io/img/in-post/2016-11-28-blend-terrain-textures/terrain_seam.png)

这主要是因为这里的纹理tiling是我们手动对worldScale取frac得到的，这样纹理采样坐标的偏导其实是不连续的，而通常我们使用单张纹理的tiling是连续的，是由图形API和硬件帮我们处理平铺类型的。

解决方法也很简单，我们只需要保证在接缝处的偏导连续不突变即可，这可以靠支持4个参数的[tex2D函数](http://http.developer.nvidia.com/Cg/tex2D.html)来解决。完整的代码如下：

```
float blendRatio = tex2D(_BlendTex, i.uv).z;

float2 worldScale = (worldPos.xz * _BlockScale);
float2 worldUv = 0.234375 * frac(worldScale) + 0.0078125;
float2 dx = clamp(0.234375 * ddx(worldScale), -0.0078125, 0.0078125);
float2 dy = clamp(0.234375 * ddy(worldScale), -0.0078125, 0.0078125);

float2 uv0 = worldUv.xy + decodedIndices.xy;
float2 uv1 = worldUv.xy + decodedIndices.zw;
// Sample the two texture
float4 col0 = tex2D(_BlockMainTex, uv0, dx, dy);
float4 col1 = tex2D(_BlockMainTex, uv1, dx, dy);
// Blend the two textures
float4 col = lerp(col0, col1, blendRatio);
```

其实就是手动算了下采样坐标worldScale的ddx和ddy，这也是为什么之前每个block要向每边拉伸了0.0078125个单位，这样才不会采样越境。上面在算ddx和ddy的时候，还把结果截取到（-0.0078125，0.0078125）即（1/128，-1/128）之间，我猜想这是为了在摄像机距地面非常的远的时候（此时ddx和ddy的绝对值会比较大，纹素密度很大），如果ddx或ddy的绝对值超过了拉伸值0.0078125（1/128），就会在接缝处采样到隔壁的block，所以要在这里使用clamp截取一下范围，下图显示了截取范围前后的区别。在此需要感谢评论区的jim童鞋，我之前只考虑到了正数的情况，没有考虑负值，这是不正确的（额这么说来其实某个上线游戏里也是不对的…）。

![terrain_clamp](http://candycat1992.github.io/img/in-post/2016-11-28-blend-terrain-textures/terrain_clamp.png)

# 写在最后

这里只是主纹理采样，当然还可以加上法线的采样，比如：

```
// Sample the two normal
fixed3 norm0 = UnpackNormal(tex2D(_BlockNormalTex, uv0, dx, dy));
fixed3 norm1 = UnpackNormal(tex2D(_BlockNormalTex, uv1, dx, dy));
// Blend the two normals
fixed3 norm = lerp(norm0, norm1, blendRatio);
norm = normalize(norm);
```

还有很多自定义的纹理可以靠这种方法来类推。另外，这种方法显然要实现一套自定义的刷地形工具给美术用。总结来看，这种方法需要的基本采样次数是：一次对BlendTex的采样，两次BlockMainTex的采样，共3次来完成9种地形纹理的混合（其实每次只能同时混合两张）。





