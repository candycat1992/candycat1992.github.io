---
layout:     post
title:      "【Unity Shader】消除纹理重复感的两种方法"
subtitle:   "Unity Shader"
date:       2016-11-29 00:00:01
author:     "Candycat"
header-img: "img/in-post/2016-11-29-texture-repetition/texture_repetition_title.jpg"
tags:
    - Game Art Tricks
---

# 写在前面

这篇文章和[【Unity Shader】地形纹理合并](http://candycat1992.github.io/2016/11/28/blend-terrain-textures/)属于一个系列的，当时对tex2D四个参数的ddx和ddy理解不太清楚，就去网上搜了下，结果发现了[iq大大的这篇文章](http://www.iquilezles.org/www/articles/texturerepetition/texturerepetition.htm)。这里简单讲下对<code>tex2D(sampler2D samp, float2 uv, float2 dx, float2 dy)</code>第3、4个参数dx和dy的理解，这两个参数在缺省时其实等同于<code>ddx(uv)</code>和<code>ddy(uv)</code>，它们是为了告诉GPU屏幕上这个像素旁边的那个像素（包括水平的和竖直方向上的相邻像素）采样的是哪个纹素，从而可以在算filter的时候正确进行混合。ddx和ddy主要是依靠GPU光栅化的时候是一个grid一个grid得一起执行的，所以可以知道相连像素之间的偏导。

好了，现在说下[iq这篇文章](http://www.iquilezles.org/www/articles/texturerepetition/texturerepetition.htm)的目的。我们都知道当把纹理的Wrap Mode设成Repeat的时候，当tiling系数较大的时候经常会看起来很假：

![texture_normal](http://candycat1992.github.io/img/in-post/2016-11-29-texture-repetition/texture_normal.png)

很明显地可以看出来有固定的pattern。这主要是因为每个0~1的tile内的纹理都是完全一样的，iq提出了两种方法来改良，使得看起来不会这么有重复感。当然，这两种技术都会成倍增大采样的次数，同时也有一些额外的计算，但效果还是不错的，在能够承受这种cost的时候还是很值得一用的。

# 方法一

这种方法的原理是避免tile都使用同样的纹理，而是对每个tile使用的纹理进行随机的翻转和平移，使得每个tile互不相同，然后在tile边界处进行混合，消除接缝问题。这里面主要涉及到了三个技术：

* 消除每个tile的重复感。这是通过判断当前所处的tile（这可以利用<code>floor(uv)</code>轻松得到），然后给每个tile一个四维的伪随机数，xy表示该tile的翻转方向（即水平和竖直方向上是否要进行mirror），zw表示该tile的平移方向，至此就可以保证每个tile都是不同的了。
    ![texture_tech1_seam_ddxddy](http://candycat1992.github.io/img/in-post/2016-11-29-texture-repetition/texture_tech1_seam_ddxddy.png)
* 上一步的结果有两个问题，首先是在tile和tile相交处有明显的接缝问题。这个可以通过算该tile旁边的三个tiles的采样结果，然后靠uv的小数部分判断距离接缝处的距离，并据此来混合四个采样结果，模糊接缝处使得结果看起来比较自然。何时开始混合边界可以当成一个参数来调节。
    ![texture_tech1_seam](http://candycat1992.github.io/img/in-post/2016-11-29-texture-repetition/texture_tech1_seam.png)
* 即便模糊了接缝处，还是会有一些残留的接缝。这些接缝产生的原因是因为我们这种方法会使得uv在tile的边界处产生很大的跳跃，导致mipmaping的时候也会出现跳跃。解决方法就是传递正确的ddx和ddy给tex2D函数，避免uv跳跃即可。
    ![texture_tech1](http://candycat1992.github.io/img/in-post/2016-11-29-texture-repetition/texture_tech1.png)

iq的Shadertoy上有个动态的例子可以看出来是怎么旋转平移的：

<iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/lt2GDd?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>

Unity Shader的主要代码如下：

```
fixed4 texNoTileTech1(sampler2D tex, float2 uv) {
	float2 iuv = floor(uv);
	float2 fuv = frac(uv);

	// Generate per-tile transformation
	#if defined (USE_HASH)
		float4 ofa = hash4(iuv + float2(0, 0));
		float4 ofb = hash4(iuv + float2(1, 0));
		float4 ofc = hash4(iuv + float2(0, 1));
		float4 ofd = hash4(iuv + float2(1, 1));
	#else
		float4 ofa = tex2D(_NoiseTex, (iuv + float2(0.5, 0.5))/256.0);
		float4 ofb = tex2D(_NoiseTex, (iuv + float2(1.5, 0.5))/256.0);
		float4 ofc = tex2D(_NoiseTex, (iuv + float2(0.5, 1.5))/256.0);
		float4 ofd = tex2D(_NoiseTex, (iuv + float2(1.5, 1.5))/256.0);
	#endif

	// Compute the correct derivatives
	float2 dx = ddx(uv);
	float2 dy = ddy(uv);

	// Mirror per-tile uvs
	ofa.zw = sign(ofa.zw - 0.5);
	ofb.zw = sign(ofb.zw - 0.5);
	ofc.zw = sign(ofc.zw - 0.5);
	ofd.zw = sign(ofd.zw - 0.5);

	float2 uva = uv * ofa.zw + ofa.xy, dxa = dx * ofa.zw, dya = dy * ofa.zw;
	float2 uvb = uv * ofb.zw + ofb.xy, dxb = dx * ofb.zw, dyb = dy * ofb.zw;
	float2 uvc = uv * ofc.zw + ofc.xy, dxc = dx * ofc.zw, dyc = dy * ofc.zw;
	float2 uvd = uv * ofd.zw + ofd.xy, dxd = dx * ofd.zw, dyd = dy * ofd.zw;

	// Fetch and blend
	float2 b = smoothstep(_BlendRatio, 1.0 - _BlendRatio, fuv);

	return lerp(	lerp(tex2D(tex, uva, dxa, dya), tex2D(tex, uvb, dxb, dyb), b.x),
					lerp(tex2D(tex, uvc, dxc, dyc), tex2D(tex, uvd, dxd, dyd), b.x), b.y);
}
```

# 方法二

这种方法更加复杂一点，相比于方法一是根据整齐的tile来平铺和混合，方法二依靠是[Voronoi分布](http://www.iquilezles.org/www/articles/smoothvoronoi/smoothvoronoi.htm)来划分和混合空间，这种划分的好处是混合是发生在Voronoi图上的，而不是整整齐齐的方格上，看起来可能更加自然。：

* 空间划分。整个空间还是会有若干的tile，但是会在每个tile内随机生成一个Voronoi点，每个点对应了一个纹理样式（靠随机平移来区分）。
    ![texture_tech2_voronoi](http://candycat1992.github.io/img/in-post/2016-11-29-texture-repetition/texture_tech2_voronoi.png)
* 混合。计算每个像素所在的周围9个Voronoi点，采样得到它们的纹理颜色，混合的时候依靠该像素到每个Voronoi点的距离的高斯衰减值作为混合权重，也就是说，距离Voronoi点越近权重越高。与方法一不同，这种方法其实随时随地都在混合（方法一的混合只发生在边界处），因此采用高斯衰减的好处就在于越靠近高斯衰减权重会迅速升高，使得混合不会造成整体非常模糊。最后，还需要对总体混合权重进行一次归一化，防止颜色失真。
    ![texture_tech2](http://candycat1992.github.io/img/in-post/2016-11-29-texture-repetition/texture_tech2.png)

iq的Shadertoy上还有个动态的例子：

<iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/4tsGzf?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>

Unity Shader的主要代码如下：

```
fixed4 texNoTileTech2(sampler2D tex, float2 uv) {
	float2 iuv = floor(uv);
	float2 fuv = frac(uv);

	// Compute the correct derivatives for mipmapping
	float2 dx = ddx(uv);
	float2 dy = ddy(uv);

	// Voronoi contribution
	float4 va = 0.0;
	float wt = 0.0;
	float blur = -(_BlendRatio + 0.5) * 30.0;
	for (int j = -1; j <= 1; j++) {
		for (int i = -1; i <= 1; i++) {
			float2 g = float2((float)i, (float)j);
			#if defined (USE_HASH)
				float4 o = hash4(iuv + g);
			#else
				float4 o = tex2D(_NoiseTex, (iuv + g + float2(0.5, 0.5))/256.0);
			#endif
		    // Compute the blending weight proportional to a gaussian fallof
			float2 r = g - fuv + o.xy;
			float d = dot(r, r);
			float w = exp(blur * d);
			float4 c = tex2D(tex, uv + o.zw, dx, dy);
			va += w * c;
			wt += w;
		}
	}

	// Normalization
	return va/wt;
}
```

# 写在最后

从性能上来看，方法一需要采样4次（如果使用噪声纹理来模拟随机数的话还要再加上4次），方法二需要采样9次（如果使用噪声纹理来模拟随机数的话还要再加上9次），当然还有一些额外的计算，所以使用的时候要权衡。

从PVRShaderEditor来看，原始采样、方法一、方法二估测出来的cycles数目大约是28、188、360。从效果上来看，原始采样 < 方法一 < 方法二的效果，尤其是在图片本身就有一定四四方方的样式，比如：

![more_results](http://candycat1992.github.io/img/in-post/2016-11-29-texture-repetition/more_results.jpg)