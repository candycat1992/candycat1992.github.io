---
layout:     post
title:      "Smoke材质的二三事"
subtitle:   "Game Art Tricks"
date:       2017-07-14 00:00:02
author:     "Candycat"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Game Art Tricks
---

## 前言

这篇文章是我第一次使用[Prose](http://prose.io/)来写博客，尝试一下。以前我都是在Cmd Markdown里写，然后编辑一下再发布的，感觉过程略蠢……

-----------------

这几天看了些关于粒子特效材质的文章，主要是怎么得到比较流畅的动画效果，比如爆炸之类的。感觉有一些想法很有意思，这篇主要参考了Simon的[Fallout 4 – The Mushroom Case](https://simonschreibt.de/gat/fallout-4-the-mushroom-case/)一文。

## 帧动画

效果基本上是基于帧动画作为基础。Simon解释了Fallout 4里面在帧动画基础上所做的一些改进。与传统的把全部信息（颜色和透明度）存储在一张帧动画图片中不同，Fallout 4里面**分离出来了颜色信息和一定程度上的透明度信息**，然后在渲染的时候**通过两张ramp texture来得到真正的值**，以此来获得更高的自由度。

* 颜色：帧动画图片中只存储光照的灰度信息，而真正的颜色可以靠一张ramp texture去决定。绝妙之处在于，这张ramp texture是二维的，横轴表示了光照强度的变化，而纵轴则是粒子的时间变化（lifetime）。这样的好处是，粒子在不同的lifetime里可以有不同的渐变颜色值，多张不同的ramp texture就可以得到一些差异较大的粒子效果，得以重用。

![smoke_remap_color](http://candycat1992.github.io/img/in-post/2017-07-14-particle-material/smoke_remap_color.png)

* 透明度：Fallout 4里部分抽离了透明度信息。帧动画的A通道仍然保存了一个透明度值，但它并不是最后的透明度，我们同样用一张二维的ramp texture来重映射透明度值，使之受lifetime的影响。

![smoke_remap_alpha](http://candycat1992.github.io/img/in-post/2017-07-14-particle-material/smoke_remap_alpha.png)

实现起来还是很简单的，主要的代码就是：

```
float3 curRampCol = tex2D(_ColorRamp, float2(curCol.r, lifeTime)).rgb;
float curRampAlpha = tex2D(_AlphaRamp, float2(curCol.a, lifeTime)).a;
```

上面的lifeTime用来控制纵坐标方向上的采样。

## Motion Vectors

后来搜资料的时候又发现了一种优化动画过渡的方法。一般帧动画的播放方法有两种，要么每时刻只播放一帧，这种一般播放速度很快并且帧与帧之间的差别不大，使人看不出跳跃；要么就是使用线性混合的方法混合相邻两帧。虽然后面这种的确可以起到一定平滑过渡的作用，但是如果相邻两帧图像差别较大，速度不够快的时候还是可以看到比较明显的穿帮。于是就有了motion vector的方法。

Klemen Lozar在[他的一篇文章](http://www.klemenlozar.com/frame-blending-with-motion-vectors/)中比较详细地讲了如何在帧动画中应用motion vector。文章中指出，这种做法最开始是由Guerrilla Games在Killzone 2的cutscene中开始使用的技术。核心思想就是靠motion vector来偏移坐标。Motion vector其实和flow map等这些概念本质上是一样的，它表明了当前这个像素点将来的走向，据此可以达到平滑流动的效果。水渲染里面也经常使用flow map来模拟河流的流动。

在实现方法上，我们除了需要传统的帧动画图像外，还需要一张motion vector贴图。据了解，一般都是在用模拟软件比如Houdini在渲染得到帧动画的同时，渲染motion vector。如果你只有一张RGB通道的普通的帧动画，我搜到了一个小神器可以直接生成——[FacedownFX](https://www.facedownfx.com/#BuildDownloads)的Slate Editor，效果还可以。下面是一个例子：

![smoke_cloud](http://candycat1992.github.io/img/in-post/2017-07-14-particle-material/smoke_cloud.png) ![smoke_motion](http://candycat1992.github.io/img/in-post/2017-07-14-particle-material/smoke_motion.png)

额因为我用的试用版，所以生成出来的图片都有水印。除了生成motion vector（还可以很方便地查看帧动画的效果），Slate Editor还可以快速分割图片，是个不错的小工具。

代码的话也比较简单，要说的话都在代码里了：

```
///
/// 使用motion vector来混合前后两帧图像，实现平滑的帧动画效果
///
Shader "Smoke Frame" {
	Properties {
		_MainTex ("Image Sequence (Tiling)", 2D) = "white" {}
		_FlowMap ("Motion Vectors", 2D) = "white" {}
		_ColorRamp ("Color Ramp", 2D) = "white" {}
		_AlphaRamp ("Alpha Ramp", 2D) = "white" {}
		_Speed ("Speed", Range(1, 100)) = 30
		_LifeTime ("Life Time", Range(0, 1)) = 0
		_DistortionStrength ("Distortion Strength", Range(0, 0.005)) = 0
	}

	SubShader{
		Tags{ "Queue" = "Transparent" "RenderType" = "Transparent" }

		Pass {
			Blend SrcAlpha OneMinusSrcAlpha
			ZWrite Off
			Cull Off

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag

			#include "UnityCG.cginc"

			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _FlowMap;
			sampler2D _ColorRamp;
			sampler2D _AlphaRamp;
			float _Speed;
			float _LifeTime;
			float _DistortionStrength;

			struct a2v {
				float4 vertex : POSITION;
				float2 texcoord : TEXCOORD0;
			};

			struct v2f {
				float4 pos : SV_POSITION;
				float4 uv : TEXCOORD0;
				float2 blendFactor : TEXCOORD1;
			};

			// 算第frame帧图像对应的uv坐标，subImages是帧图像的行列数目
			inline float2 SubUVCoordinates(float2 uv, float frame, float2 subImages) {
				float time = floor(frame);
				float row = floor(time / subImages.y);
				float column = time - row * subImages.x;
				uv = uv + half2(column, -row);

				uv.x /= subImages.x;
				uv.y /= subImages.y;

				return uv;
			}

			v2f vert (a2v v) {
				v2f o;

				o.pos = UnityObjectToClipPos(v.vertex);

				float frame = _Time.y * _Speed;
               			// 前后两帧图像的uv坐标
				o.uv.xy = SubUVCoordinates(v.texcoord.xy, frame, _MainTex_ST.xy);
				o.uv.zw = SubUVCoordinates(v.texcoord.xy, frame + 1, _MainTex_ST.xy);
                		// 混合前后量帧时使用的混合系数
				o.blendFactor.x = frac(frame);
                		// 计算lifetime，第0帧对应lifetime值0，最后一帧对应lifetime值1
				o.blendFactor.y = frac(frame / (_MainTex_ST.x * _MainTex_ST.y));

				return o;
			}

			inline float2 SampleMotionVector(float2 uv) {
            			// 这里会乘以float2(1, -1)是因为motion vector中存储的方向和Unity里面纹理采样的方向有所不同
				return (tex2D(_FlowMap, uv).rg * 2.0 - 1.0) * float2(1, -1);
			}

			float4 frag (v2f i) : SV_Target {
				float2 curUV = i.uv.xy;
				float2 nextUV = i.uv.zw;
				float blendFactor = i.blendFactor.x;
				float lifeTime = i.blendFactor.y;

				float2 curMotion = SampleMotionVector(curUV);
				float2 nextMotion = SampleMotionVector(nextUV);

				// 使用当前帧/后一帧的motion vector偏移当前帧/后一帧的采样坐标
				curUV = curUV - curMotion * blendFactor * _DistortionStrength;
				nextUV = nextUV + nextMotion * (1 - blendFactor) * _DistortionStrength;

				float4 curCol = tex2D(_MainTex, curUV);
				float4 nextCol = tex2D(_MainTex, nextUV);

				// 使用前一节的方法映射得到真正的颜色值和透明值
				float3 curRampCol = tex2D(_ColorRamp, float2(curCol.r, lifeTime)).rgb;
				float curRampAlpha = tex2D(_AlphaRamp, float2(curCol.a, lifeTime)).a;
				float3 nextRampCol = tex2D(_ColorRamp, float2(nextCol.r, lifeTime)).rgb;
				float nextRampAlpha = tex2D(_AlphaRamp, float2(nextCol.a, lifeTime)).a;

				float4 col;
				col.rgb = lerp(curRampCol, nextRampCol, blendFactor);
				col.a = lerp(curRampAlpha, nextRampAlpha, blendFactor);

				return col;
			}

			ENDCG
		}
	}
}
```

## 过程式生成的Smoke效果

Simon的文章很有意思的一点是，他总是会举一反三列举很多相关的实现链接。[Fallout 4 – The Mushroom Case](https://simonschreibt.de/gat/fallout-4-the-mushroom-case/)这篇文章的Update 4里列举了一个方法，实现的效果很有趣：

![Smoke09](https://data.simonschreibt.de/gat056/update4/Smoke09.gif)

然后我就尝试按照[原文](http://www.zoltane.com/pages/unreal/drone-alone/smoke-material/)中的实现了一下。这个作者Zoltan是一名TA，之前就看过他写的一些很有意思的文章，但是每次看的时候没有辣么瘾就是因为他对于技术实现总是写的非常简略，不像其他博主起码会给出一些代码片，所以要重现他的效果要完全自己猜。然后我重现的效果是……

![smoke_particle](http://candycat1992.github.io/img/in-post/2017-07-14-particle-material/smoke_particle.png)

不谈效果好坏吧，我觉得实现的过程还是很有意思。他的做法也应用到了motion vector的思想，通过在cellular noise上应用motion vector控制噪声流动，来模拟一种很有意思的孔洞变大的效果。我先放上代码吧，关于噪声真的很有意思，有时间在下一篇里面再写吧。


```
Shader "Smoke Particle" {
	Properties {
		_ParticleNoiseTex ("Noise Tex (RG: Derivatives B: Noise)", 2D) = "white" {}
		_RampTex ("Ramp Tex", 2D) = "white" {}
		_NoiseTex ("Noise Tex", 2D) = "white" {}
		_Speed ("Speed", Range(0, 5)) = 1
	}

	SubShader{
		Tags{ "Queue" = "Transparent" "RenderType" = "Transparent" }

		Pass {
			Blend SrcAlpha OneMinusSrcAlpha
			ZWrite Off
			Cull Off

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag

			#include "UnityCG.cginc"

			sampler2D _ParticleNoiseTex;
			float4 _ParticleNoiseTex_ST;
			sampler2D _RampTex;
			sampler2D _NoiseTex;
			float _Speed;

			struct a2v {
				float4 vertex : POSITION;
				float2 texcoord : TEXCOORD0;
			};

			struct v2f {
				float4 pos : SV_POSITION;
				float4 uv0 : TEXCOORD0;
				float2 uv1 : TEXCOORD1;
			};

			v2f vert (a2v v) {
				v2f o;

				o.pos = UnityObjectToClipPos(v.vertex);

				o.uv0.xy = TRANSFORM_TEX(v.texcoord, _ParticleNoiseTex);
				o.uv0.zw = TRANSFORM_TEX(v.texcoord, _ParticleNoiseTex).yx + float2(0.5, 0.5);

				o.uv1.xy = v.texcoord;

				return o;
			}
			
			inline float SampleNoise(float2 uv, float dist) {
				float2 derivatives = tex2D(_ParticleNoiseTex, uv).rg * 2.0 - 1.0;
				float noise = tex2D(_ParticleNoiseTex, uv + derivatives * float2(1, -1) * dist).b;
				return noise;
			}

			inline float RadialGradient(float2 pos, float radius, float blur) {
				return 1.0 - smoothstep(1.0 - blur * 2.0, 1.0, sqrt(dot(pos, pos) / max(0.0001, radius * radius)));
			}

			float4 frag (v2f i) : SV_Target {
				float2 dist = i.uv1.xy - 0.5;
                		// 来模拟从中间向外扩散的效果
				float time = dot(dist, dist) * (0.5 + tex2D(_ParticleNoiseTex, i.uv1.xy * 0.3).b * 0.5) * 4.0 - _Time.y * _Speed;

				// 模拟两层运动
				float loop1 = time;
				float loop2 = time + 0.5;
				float2 offsetUV0 = float2(floor(loop1) * 0.01, 0.3);
				float2 offsetUV1 = float2(floor(loop2) * 0.01, 0.6);
				float distort0 = frac(loop1);
				float distort1 = frac(loop2);

				float2 uv0 = i.uv0.xy + tex2D(_NoiseTex, offsetUV0).rg;
				float2 uv1 = i.uv0.zw + tex2D(_NoiseTex, offsetUV1).rg;
				float noise0 = SampleNoise(uv0, lerp(-0.05, 0.05, distort0));
				float noise1 = SampleNoise(uv1, lerp(-0.05, 0.05, distort1));

				float noise = lerp(noise0, noise1, lerp(0, 1, abs(distort0 - 0.5) * 2.0));
				noise = pow(noise, 1.2);

				// 计算smoke的颜色
				float fieryFade = lerp(0.5, 1.5, RadialGradient(dist, 0.2 + noise * 0.3, 0.5));
				float3 col = tex2D(_RampTex, float2(noise, 0.5)) * fieryFade;
				float alpha = RadialGradient(dist, 0.2 + noise * 0.3, 0.1);

				return float4(col, alpha);
			}

			ENDCG
		}
	}
}

```


