---
layout:     post
title:      "「Graphics Study」RDR2渲染分析 —— 阴影篇（上）"
subtitle:   ""
date:       2021-12-12 00:49:02
author:     "Candycat"
header-img: "img/in-post/rdr2-bg.png"
tags:
    - Graphics Study
---

## 前言

12月12日是这篇博客的预期发布时间，没错我鸽了一周，额高估了自己低估了截帧的复杂度，程序员预估的deadline果然容易过分乐观--||

---

回到正题，我一直感叹RDR2的阴影渲染质量很高，数毛社有篇[分析视频](https://youtu.be/Dnzuh6I8gnM?list=PLrL3xbgUaxxjBKisD282x6YbGIGwowTCq&t=520)演示了RDR2的阴影表现，没玩过的小伙伴一定不要错过。RDR2的阴影有非常出色的Contact Shadow，即距离物体更近的阴影更加锐利，反之越远越模糊。这个模糊半径甚至和当前的天气状况有关，可谓丧心病狂：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/contact-shadow0.png" alt="contact-shadow0" style="zoom:60%;" />  <img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/contact-shadow1.png" alt="contact-shadow1" style="zoom:60%;" />

因此这一篇我们就主要分析来RDR2是如何绘制平行光阴影的。其实本来想一篇博客就写完的，但没想到内容有点多，就分为上下篇吧。

---

先回忆[上一篇](http://candycat1992.github.io/2021/12/09/rdr2-study/)的内容，我们给出了GBuffer的Layout，其中跟阴影绘制相关的主要是GBufferC的A通道，它包含了逐材质计算的一些平行光阴影信息（例如在计算Parallax Mapping时计算的自阴影信息），这也是平行光阴影的起点。RDR2后续会继续计算屏幕空间阴影、CSM阴影等，将它们结合起来作为最终的平行光阴影更新到GBufferC的A通道：

|                      GBufferC.a Before                       |                       GBufferC.a After                       |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| <img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/gbufferc_a.jpg" alt="csm" style="zoom: 50%;" /> | <img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/shadow-final.jpg" alt="shadow-final" style="zoom:50%;" /> |

可以看到，RDR2最后得到的平行光阴影非常柔和，它的半影范围很大，甚至可以媲美Ray Traced Shadow。RDR2为了得到这样的效果也做了很多事情。总体来说，RDR2绘制平行光阴影包括几个计算部分：

* 处理Scene Stencil，标记出边界像素部分，以便在后面的Shadow Pass里对边界像素计算抗锯齿后的阴影（可选）
* 绘制场景的Cascade Shadow Map（CSM）
* 处理上一步的CSM，为CSM每一级计算一定半径范围的最小/最大深度值，以便后面计算软阴影
* 计算平行光阴影
  * 绘制远距离阴影
  * 绘制近距离阴影


其中，最后一个部分中绘制近距离阴影的计算我们放到下一篇讲。下面我们就来具体分析上述与阴影相关的各个Pass。

## 处理Scene Stencil

一开始在场景的GBuffer绘制完成后，初始Stencil Buffer大致如下：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/stencil-input.jpg" alt="stencil-input" style="zoom:50%;" />

在开始渲染CSM之前，RDR2会处理上述的Scene Stencil Buffer。总体来说，这些处理的目的是对GBuffer的各个属性进行边缘检测，将有差异的边缘部分在Stencil Buffer中标记出来（对应Stencil的第6个bit），之后会靠这些标记为屏幕边界像素的计算抗锯齿后的阴影。这个标记处理可以分为两个Screen Pass。

### Screen Pass 0：标记Stencil的差异部分

第一个Screen Pass的输入就是Stencil Buffer本身。由于RDR2开启了8x MSAA来渲染GBuffer，因此在这个Pixel Shader里可以为每个Pixel手动采样8x MSAA的8个samples，分析它们的Stencil值的差异，据此来计算边缘检测。这个Pass的伪代码大致如下：

```c++
pixel_shader (every pixel with screen position (x, y))
{
    int2 Index = int2(x, y);
    int2 Offset = int2(0, 0);
    
    int StencilOr = 0;
    int StencilAnd = 0xFF;
    for (int i = 0; i  < 7; i++)
    {
        int Stencil = StencilTexture.Load(Index, Offset, i);
        StencilOr |= Stencil;
        StencilAnd &= Stencil;
    }

    if (StencilOr & 0x20)
        return 1;
    else if ((StencilOr ^ StencilAnd) & (-33))
        return 1;
    else
        return 0;
}
```

其实本质来说，上面Pass的结果就是判断8个MSAA samples的Stencil值是否完全一样，如果完全一致就输出黑色，否则就标记它为一个特殊像素。如果其中任意一个sample的Stencil值已经被标记成了0x20这个Bit（即已经被标记为特殊的边界像素了），就直接保留它。

这个Pass的计算结果会输出到一张格式为R8_UNORM的权重图中（为显示明显对下图进行了提亮）：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/stencil-mask.png" alt="stencil-mask" style="zoom:50%;" />

注意到上图中大部分标红区域相当于Stencil的边缘检测结果，而大片的红色区域（似乎是某些特定的墙壁和灌木部分）就对应了Stencil & 0x20的部分。

### Screen Pass 1：标记GBuffer的差异部分

第二个Screen Pass会对Stencil Buffer进行真正的标记。整个Pass会利用Stencil Test忽略那些Stencil已经被标记为0x20的部分，而只修改其余部分像素的Stencil。下图显示了这个Pass的Stencil Test结果（红色为Stencil & 0x20部分，绿色为!(Stencil & 0x20)部分）：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/stencil-0x20.png" alt="stencil-0x20" style="zoom:50%;" />

上述绿色的屏幕像素部分的Stencil值，会在这个Pass中被继续修改。简单来说这个Pass的目的是进一步检测MSAA各个samples之间的GBuffer数据是否相同，如果不同则标记成一个边界像素。再结合上一个Pass标记出来的Stencil边界像素部分，把这些所有的边界像素部分统一标记为0x20。伪代码大致如下：

```c++
pixel_shader (every pixel with screen position (x, y))
{
    int2 Index = int2(x, y);
    int2 Offset = int2(0, 0);

	bool bStencilIsSame = DecodeStencilMask(StencilMask.Load(Index, Offset)) == 0;

	bool bGBufferIsDiff = false;
	int2 CompPairs[4] = { int2(7, 5), int2(5, 6), int2(6, 0) };	
	for (int i = 0; i < 4; i++)
	{
		int2 SamplePair = CompPairs[i];
		FGBufferData SampleData0 = DecodeGBufferData(Index, Offset, SamplePair.x);
		FGBufferData SampleData1 = DecodeGBufferData(Index, Offset, SamplePair.y);
		bGBufferIsDiff |= CheckGBufferDataIsDiff(SampleData0, SampleData1);
	}

	if (!bGBufferIsDiff && bStencilIsSame) discard;
}
```

上面的代码在比对GBuffer数据时共采样了3对samples，这3对samples的位置关系可以参考[Microsoft的文档](https://microsoft.github.io/DirectX-Specs/d3d/archive/D3D11_3_FunctionalSpec.htm)：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/sample-pattern.png" alt="sample-pattern" style="zoom: 80%;" />

判定使用的GBuffer数据包括Depth、GBufferB（Normal）、GBuferC的yz通道（猜测这两个通道编码了计算Specular使用的材质信息）。如果各个samples之间的Stencil值和GBuffer值被判定为相同（实际代码里会检测一定的误差判定范围），这个pixel就会被discard。只有那些有差异的像素会得以保留来更新Stencil的值。

---

经过两个Pass的处理后，标记前后的Stencil Buffer对比如下：

| Stencil Before | Stencil After |
| :---: | :---: |
|<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/stencil-input.jpg" alt="stencil-input" style="zoom:50%;" />|<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/stencil-output.jpg" alt="stencil-output" style="zoom:50%;" />|

可以发现，现在所有的边界像素都在Stencil Buffer中被标记了出来。

## 绘制CSM

平行光的Shadowmap使用了常见的CSM策略。RDR2共使用了四级CSM，每一级分辨率为2048x2048，总共分辨率为2048x8192：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/csm.png" alt="csm" style="zoom:80%;" />

### 绘制电线的Mask和Shadowmap

RDR2对于电线这种很细的物体的CSM是单独另开Pass进行绘制的。除了绘制到上面的CSM中，还额外分配了一张格式为A8_UNORM、大小为2048x8192的纹理作为Color RT，这张RT记录了电线的Mask信息：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/wires.png" alt="wires" style="zoom:80%;" />

这张Mask会在后面计算CSM阴影的时候用到。

## 处理CSM计算最小/最大深度值

这部分计算的主要目的是为CSM的每一级Shadowmap分别计算不同半径范围内的最小/最大深度值，将结果保存到另一张RT里，以便在后续的Pass里计算软阴影。这部分计算可以再细分为以下两个部分。

### 初始化四分之一分辨率的深度图

绘制完整个场景的CSM后，RDR2会根据它再生成两张四分之一分辨率（512x2048）、格式均为R16G16的RTs：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/shadow_init.png" alt="shadow_init" style="zoom:80%;" />

这两张RT分别包含了：

* RT0：似乎是计算了四分之一分辨率下的VSM
* RT1：为四分之一分辨率下的每个输出像素，计算其对应在全分辨率CSM下、每个4x4块中的最小深度值和最大深度值，分别存储到RG通道中

> 由于上面的RT0和场景的平行光阴影没有直接关系，这里我们就不再讨论。这个RT0的作用主要是作为一组Compute Shader的输入来计算得到一张3D Texture，似乎是给后续计算God Ray等效果使用的，之后有机会再讨论吧。

计算RT1部分的伪代码如下：

```c++
compute_shader (every pixel in RT1 with position (x, y))
{
	int2 OutputIndex = int2(x, y);
    float2 BufferUV = (OutputIndex + 0.5) / TextureSize;

	float MinDepth = FLT_MAX;
	float MaxDepth = FLT_MIN;
	for (int i = {-1, 1})
	{
		for (int j = {-1, 1})
		{
			float4 ShadowDepths = DepthTexture.Gather(BufferUV, int2(i, j));
			MinDepth = min(MinDepth, min(min(ShadowDepths.y, ShadowDepths.w), min(ShadowDepths.x, ShadowDepths.z)));
			MaxDepth = max(MaxDepth, max(max(ShadowDepths.y, ShadowDepths.w), max(ShadowDepths.x, ShadowDepths.z)));
		}
	}

	Output[OutputIndex] = float2(MinDepth, MaxDepth);
}
```

通过4次Gather计算，RT1的每个像素可以计算在全分辨率CSM下该点周围半径2个像素大小范围（共16个有效像素）内的最小深度值和最大深度值。

### 计算更大范围的最小/最大深度值

RDR2使用了更多的Pass去计算更大半径范围的最小和最大深度值。这个部分包含了4个Compute Pass，每个Pass负责处理初始化Pass中输出的RT1（即四分之一分辨率下的最小/最大深度值）中的某一级Cascade，为其计算一定半径内阴影深度的最大和最小值，并将结果存储到另一张512x2048的RT里。这部分伪代码如下：

```c++
compute_shader (every pixel in Cascade0/1/2/3 in RT1 with position (x, y))
{
	int2 OutputIndex = int2(x, y);

	float MinDepth = FLT_MAX;
	float MaxDepth = FLT_MIN;
	for (int i = -SearchRadius; i <= SearchRadius; i++)
	{
		int SubSearchRadius = floor(sqrt(SearchRadius * SearchRadius - i * i));
		for (int j = -SubSearchRadius; j <= SubSearchRadius; j++)
		{
			float2 MinMaxDepth = RT1.Load(int2(j, i)).xy;

			MinDepth = min(MinDepth, MinMaxDepth.x);
			MaxDepth = max(MaxDepth, MinMaxDepth.y);
		}
	}

	Output[OutputIndex] = float2(MinDepth, MaxDepth);
}
```

对于每一级Cascade来说，上面的SearchRadius是不同的：

* Cascade 0：SearchRadius = 8（对应全分辨率CSM下的半径32个像素），采样了197次纹理
* Cascade 1：SearchRadius = 4（对应全分辨率CSM下的半径16个像素），采样了49次纹理
* Cascade 2：SearchRadius = 3（对应全分辨率CSM下的半径12个像素），采样了29次纹理
* Cascade 3：SearchRadius = 0（对应全分辨率CSM下的半径2个像素），采样了1次纹理

可见，RDR2对Cascade 0计算的半径是非常可怕的（性能也很可怕），也难怪可以做出半影范围那么大的软阴影了。

经过这4个CS计算后，最终得到每一级Cascade一定半径范围内的最小深度值和最大深度值：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/shadow-dilation-erosion.png" alt="shadow-dilation-erosion" style="zoom:80%;" />

## 计算平行光阴影

RDR2的平行光阴影共包括两个部分：

* 不使用CSM的远距离阴影
* 使用CSM的近距离阴影

这两种类型的阴影会通过设置不同的[Depth Bounds](https://microsoft.github.io/DirectX-Specs/d3d/DepthBoundsTest.html)来处理不同距离的阴影。其中，远距离阴影范围大约覆盖距离摄像机深度值>200米的区域，近距离阴影覆盖距离摄像机深度值<200米的区域。每种类型的阴影绘制会再细分到2个Screen Pass中（共4个Screen Pass），这2个Screen Pass会基于之前处理得到的Stencil Buffer、使用Stencil Test来处理屏幕空间的不同像素部分，第一个Screen Pass处理绝大部分常规像素，第二个Screen Pass处理之前被特殊标记的那些Stencil值或GBuffer值有差异的边界像素部分。这两个Screen Pass的Stencil Test通过结果如下所示：

|                        Screen Pass 0                         |                        Screen Pass 1                         |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| <img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/screen-pass0-stencil-test.png" alt="screen-pass0-stencil-test" style="zoom: 50%;" /> | <img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/screen-pass1-stencil-test.png" alt="screen-pass1-stencil-test" style="zoom:50%;" /> |

这两个Screen Pass其实代码基本完全相同，只是Screen Pass 0在采样GBuffer（包括Depth&Stencil Buffer）时直接采样SampleIndex=0的位置，而Screen Pass 1的Pixel Shader会额外传入MSAA的Sample Index，使用这个Index再去采样GBuffer（包括Depth&Stencil Buffer）进行相关计算。原因在于，我们之前提到过Shadow Pass的Color RT其实是GBufferC，而RDR2中的GBuffer都会开启MSAA，因此Screen Pass 1可以利用这一特性在边界像素处手动计算各个MSAA Sample位置处的阴影结果，相当于在边界处手动计算了阴影的SSAA抗锯齿。但代价是原本只需要执行一次的Pixel Shader在Screen Pass 1里要执行8次，这也是为什么一开始RDR2要把这些边界像素单独标记出来。

>这里利用了DirectX的SV_SampleIndex语义，具体可参见[Microsoft的文档](https://docs.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics)。MJP也写过[相关文章](https://mynameismjp.wordpress.com/2015/09/13/programmable-sample-points/)讲解过可编程的MSAA特性，推荐阅读。

由于两个Screen Pass的代码几乎完全一样，区别只在于是否需要单独采样MSAA的Sample Index处理抗锯齿，因此我们下面只解释每种类型阴影计算的具体原理，不再赘述这两个Screen Pass的区别了。

> 实际上，每种阴影类型是否需要再细分到两个Screen Pass似乎是由摄像机位置和渲染质量决定的，在低配或者离地角度比较高的时候，RDR2就不会再拆分这两个Screen Pass，而是直接使用一个Screen Pass绘制所有远/近距离像素了，不再靠Stencil去单独处理边界像素的阴影了，这样一共只需要两个Pass去绘制全屏幕的阴影。

每种距离的阴影计算来源不同的，我们先来看远距离阴影的计算部分。

### 绘制远距离阴影

我们之前选取的这一帧截图由于远处大部分区域被房屋遮挡住了，看不太出来远距离阴影的变化，因此这里我们临时换成另一帧远距离阴影计算前后变换更明显的图像进行说明：

| Far Shadow Before | Far Shadow After |
| :---: | :---: |
|<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-shadow-before.png" alt="far-shadow-before" style="zoom:50%;" />|<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-shadow-after.png" alt="far-shadow-after" style="zoom:50%;" />|

可以看到，远距离阴影主要有以下几个计算来源：

* 体积云阴影
* 屏幕空间阴影
* 烘焙阴影
* 地形阴影

这些阴影的计算都不依赖CSM，而是使用其他的数据计算实现。

#### 云的阴影

云的阴影计算比较容易理解，主要还是依赖Shadowmap。RDR2为体积云渲染了另一张Shadowmap：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/cloud-shadowmap.png" alt="cloud-shadowmap" style="zoom:50%;" />

这张Shadowmap的绘制也是本帧通过CS完成的，之后有时间我再补充到这里。

通过在Pixel Shader里把当前的像素坐标转换到体积云的Shadowmap空间，再比较当前像素深度和Shadowmap中的已有深度，就可以计算得到云的阴影值。这部分伪代码如下：

```c++
pixel_shader (every pixel with screen position (x, y))
{
    int2 Index = int2(x, y);
    float Depth = DepthTexture.Load(Index);
    float3 ViewSpacePos = ConvertToViewSpacePosition(Depth);

    float Shadow = 1.0;

    // Compute cloud shadow
    if (IsWithinCloudShadowSpace(ViewSpacePos))
    {
        float2 CloudSpaceUV = ConvertToCloudSpace(ViewSpacePos);
        float CloudSpaceDepth = CloudShadowDepth.Sample(CloudSpaceUV);
        
        float CloudShadow = ComputeCloudShadow(CloudSpaceDepth, ViewSpacePos);
        float CloudShadowWeight = ComputeToCloudSpaceBorderWeight(ViewSpacePos);
        
        Shadow *= lerp(1.0, CloudShadow, CloudShadowWeight);
    }

    ...
}
```

体积云的阴影覆盖范围似乎是有限的，所以RDR2考虑了当前像素点距离覆盖边界的权重，当超过体积云阴影覆盖范围时就会退化到阴影值1。

#### 屏幕空间阴影

RDR2会在屏幕空间沿着光源方向计算一定数目的shadow trace（在截帧数据中NumTrace = 12），比较每个trace point的深度值和Scene Depth中的深度值计算屏幕空间的阴影。这部分伪代码如下：

```c++
pixel_shader (every pixel with screen position (x, y))
{
    int2 Index = int2(x, y);
    float Depth = DepthTexture.Load(Index);
    float3 ViewSpacePos = ConvertToViewSpacePosition(Depth);

    float Shadow = 1.0;
    
    ....

    // Compute screen space shadow
    int Stencil = StencilTexture.Load(Index);
    float3 Normal = DecodeWorldNormal(GBufferB.Load(Index));
    int NumTrace = GetScreenSpaceTraceCount(Depth, Stencil);

    float3 ScreenTraceStartPos = GetPixelScreenSpacePosition(Depth);
    float3 ScreenTraceEndPos = GetTraceStartPosition(Depth, Normal, LightDir);
    float3 ScreenTraceStep = (ScreenTraceEndPos - ScreenTraceStartPos) / NumTrace;

    float ScreenSpaceShadow = 1.0;
    float3 ScreenTracePos = ScreenTraceStartPos + Random * ScreenTraceStep;
    for (int i = 0; i < NumTrace; i++)
    {
        int2 TracePosIndex = floor(ScreenTracePos.xy * BufferSize);
        float TracePosDepth = DepthTexture.Load(TracePosIndex);
        ScreenSpaceShadow *= ComputeScreenSpaceShadow(TracePosDepth, ScreenTracePos.z);
        ScreenTracePos += ScreenTraceStep;
    }

    ApplyScreenSpaceShadowWeight(ScreenSpaceShadow, LightDir, Normal, Depth);
    Shadow *= ScreenSpaceShadow;

    ...
}
```

其实计算屏幕空间阴影的时候还是有很多细节处理的，比如RDR2考虑了Stencil值和是否是背光面来影响trace的距离、步数以及最终的阴影权重，这部分计算因为个人能力有限理解还不到位就不写出来误导人了。

#### 烘焙阴影

这部分计算很有意思，妙啊妙啊。RDR2应该是提前烘焙了8个方向的平行光入射角度下整个地图（覆盖大约12.5km x 12.5km）中某些大型遮挡物的阴影投影结果，把它们存储到两张分辨率为512x512、格式为R16G16B16A16的纹理中，一共有8个方向的阴影信息，绑定到Pixel Shader的Input Texture 6&7上：

| InputTexture6.r | InputTexture6.g | InputTexture6.b | InputTexture6.a |
| :---: | :---: | :---: | :---: |
|<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-input6-r.png" alt="far-input6-r"  />|![far-input6-g](http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-input6-g.png)|![far-input6-b](http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-input6-b.png)|![far-input6-a](http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-input6-a.png)|

| InputTexture7.r | InputTexture7.g | InputTexture7.b | InputTexture7.a |
| :---: | :---: | :---: | :---: |
|![far-input7-r](http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-input7-r.png)|![far-input7-g](http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-input7-g.png)|![far-input7-b](http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-input7-b.png)|![far-input7-a](http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/far-input7-a.png)|

Pixel Shader里会根据当前的光源方向计算8个方向的权重对它们的采样结果进行混合，再计算得到的真正的阴影值。

但细想一下会发现，这8个方向只能表示XY平面（Z为垂直方向）上的阴影变化，但光照的仰角变化要怎么办呢？这就是妙的地方，实际上这8个方向存的并不是绝对阴影值，而是一个仰角弧度值。我猜测这8个方向的烘焙过程是这样的（纯属猜测概不负责欢迎讨论）：给定光源在XY平面的入射方向（8张图对应了8个固定方向），逐渐改变光源的仰角，使其从最小角度逐渐变化到最大仰角角度，检查地图上每个位置此时是否处于阴影中，如果在多个角度下都处于阴影中，就记录下这些角度的最大值，最后把这个角度存储到贴图中。也就是说，这8个方向阴影图中存储的值实际上是角度（以弧度为单位）。在Pixel Shader里得到加权混合后的烘焙阴影角度后，再次根据当前光源的仰角与烘焙角度进行比较，只有当烘焙角度≥光源仰角时，才意味着该位置此刻处于阴影中。这部分计算的伪代码如下：

```c++
pixel_shader (every pixel with screen position (x, y))
{
    int2 Index = int2(x, y);
    float Depth = DepthTexture.Load(Index);
    float3 ViewSpacePos = ConvertToViewSpacePosition(Depth);

    float Shadow = 1.0;

    ...

    // Compute baked shadow
    float3 SamplePosWithinMap = ComputePositonWithinGameMap(Depth);
    float4 BakedShadowAngles0 = InputTexture6.Sample(SamplePosWithinMap.xy);
    float4 BakedShadowAngles1 = InputTexture7.Sample(SamplePosWithinMap.xy);
    float BakedShadowAngle = max(dot(BakedShadowAngles0, BlendWeights0), dot(BakedShadowAngles1, BlendWeights1);

    float BakedShadow = saturate(abs(LightPitchAngle - BakedShadowAngle) / BlendAngle);
    BakedShadow = saturate((BakedShadowAngle > LightPitchAngle) ? (smoothstep(1.0, 0.0, BakedShadow) * 0.5) : (smoothstep(0.0, 1.0, BakedShadow) + 0.5));
    Shadow *= BakedShadow;
    
    ...
}
```

可以发现上面伪代码的最后并不是直接取二值对比结果，RDR2会传入一个过渡角度来做阴影的渐变：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/baked-shadow-blend.png" alt="baked-shadow-blend" style="zoom:50%;" />

感兴趣的话可以去看下在[desmos上的一个实时演示](https://www.desmos.com/calculator/520xetuxff?lang=en)，改改参数调调看就可以理解了。

这种方法当然只是一种近似，它的可行性建立在一个重要的假设上：当固定光源XY平面角度且仰角角度从小到大变化时，地图上每个观察点的阴影变化是单调的。这一假设在充分空旷环境下绝大部分时候是成立的，但对于有复杂遮挡物的环境来说，它明显有很多无法成立的情况，所以我猜测RDR2可能烘焙的是一些比较大结构的遮挡物的阴影投影状况。

#### 地形阴影

这部分是我猜测绘制的是地形阴影，因为这部分计算主要依靠采样Pixel Shader的Input Texture 8（左图），它是一张分辨率为1024x1024、格式为R16_UNORM的纹理，看起来像是RDR2整个地图环境地形的归一化后的高度图，刚好对应了游戏地图（右图，来源[Reddit](https://www.reddit.com/r/reddeadredemption/comments/gimo7v/10mp_rdr2_game_map_redux_enhanced_with_hillshaded/)）中的山区部分：

<img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/heightmap.png" alt="heightmap" style="zoom:50%;" />   <img src="http://candycat1992.github.io/img/in-post/2021-12-12-rdr2-shadows/rdr2-map.jpg" alt="rdr2-map" style="zoom:50%;" />

这部分计算比较好理解，就是沿着光源方向、按照固定步长去trace一定数目的高度图（在截帧中NumTrace = 8），比较每次trace point的高度值和Heightmap中记录的高度值，据此计算阴影。这部分计算伪代码如下：

```c++
pixel_shader (every pixel with screen position (x, y))
{
    int2 Index = int2(x, y);
    float Depth = DepthTexture.Load(Index);
    float3 ViewSpacePos = ConvertToViewSpacePosition(Depth);

    float Shadow = 1.0;

    ...

    // Compute shadow from height map
    for (int i = 0; i < NumTrace; i++)
    {
        float3 SamplePos = SamplePosWithinMap + HeightMapTraceStep * i;
        float SampleHeight = HeightMap.Sample(SamplePos.xy);
        float HeightDiff = SampleHeight - SamplePos.z;
        Shadow *= 1.0 - saturate((HeightDiff + i * HeightBias) * MaxHeight / BlendHeight)
    }

    return Shadow;
}
```

从截帧来看，用来归一化Heightmap使用的MaxHeight大约为700~800米，BlendHeight大约为10米~20米。

### 绘制近距离阴影

写不动了，我们阴影篇（下）再见哈。
