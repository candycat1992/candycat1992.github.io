---
layout:     post
title:      "「总结」历年图形渲染相关的演讲"
subtitle:   ""
date:       2022-06-11 12:00:00
author:     "Candycat"
header-img: "img/in-post/default-bg.jpg"
tags:
    - 总结
---

## 前言

上海疫情在家快三个月了，完成了4月份立的两个flag，一个是要坚持锻炼（健身环终于一周目通关了，虽然体重基本没啥变化- -），一个要刷今年刚出的GDC。之前也一直在看但经常过段时间就有点想不起来在哪篇看到了哪个感兴趣的技术，想到索性可以把所有图形渲染相关的演讲搞到一个地方，方便检索，这就是这篇的目的了。

我的方法是，先过一遍GDC、SIGGRAPH这种会议，把跟图形渲染相关的演讲标题总结到这里，我会按照自己的理解分成几个类别，可以通过目录快速跳转，然后再重点挑自己感兴趣的过一遍。不一定完全是程序相关的，有些跟TA、PCG等我比较感兴趣的也会放进来，当然为了索引完整性还有一些水货会混进来 :)

目前更新到的会议有（主要是GDC 2022，其他不定期更新中）：
* GDC (2022, 2021)
* SIGGRAPH (2021)
* Digital Dragons (2021)

## 综合向

啥都会讲一点的那种。

### [GDC 2022] One Frame in 'Halo Infinite'

链接：[PDF](https://gdcvault.com/play/1027657/One-Frame-in-Halo-Infinite)，[Video](https://gdcvault.com/play/1027724/One-Frame-in-Halo-Infinite)

公司：343 Industries - Microsoft

### [GDC 2022] An Overview of the 'Diablo II: Resurrected' Renderer

链接：[PDF](https://gdcvault.com/play/1027558/An-Overview-of-the-Diablo)，[Video](https://gdcvault.com/play/1028027/An-Overview-of-the-Diablo)

公司：Blizzard Entertainment

简介：介绍了给Diablo II开发的新渲染器，基于Forward+（便于快速开发shader），没有使用任何烘焙光照（有大量动态效果和过程化生成的场景），非常注重效果的Scalability（要能随时开关，保证可以在Switch这样的低端机上运行）：
* Overview of the renderer：简单介绍了整个管线包含的各个pass，比如grass visibility、light bin、global attenuation等
* Deep dive into select techniques：重点挑选了几个技术点介绍，包括：
    * Specular Antialiasing：求法线的偏导得到着色表面的几何复杂度，将其加到粗糙度上减少aliasing
    * Scalable Lighting：场景光源很多走完整的forward lighting在低端机上开销太大，解决方法是维护玩家周围一定范围内的3D lightmap，把场景里的光源贡献渲染到这些voxel里。具体是使用CS生成两张32x8x32 PF16精度的纹理，一张负责累加相对于这个grid position的所有光源的方向，另一张存储光源的radiance，并使用一种近似方法来模拟高光
    * Tonemapping：基于Call of Duty的Tonemapping方法
        * tonemapping曲线几乎是线性的，主要目的就是为了把场景亮度round到10000 nits（HDR标准的peak brightness）
        * display mapping曲线是给显示器用的，和CoD一样，LDR显示器使用sRGB曲线，HDR使用BT.2390标准曲线
    * Color Grading：基于Call of Duty和Frostbite的工作
        * 以exr格式导出场景的HDR图像，其中不包含任何tonemapping和其他后处理效果
        * 再导出一张.cube格式的LUT，其中编码了LDR的tonemapping曲线
        * 导入Davinci，项目设置会使用上面导出的EXR和LUT以便让初始效果和引擎内看到的完全一致
        * 美术在Davinci里调色，然后导出格式同样为.cube的65x65x65 LUT作为这个场景的color grading
        * 使用PQ曲线去解决LUT的精度问题
    * Order Independent Transparency (OIT)：因为游戏有大量效果是半透明的，同时玩家观察距离又很近，所以对透明排序要求很高
        * 参考了weighted blended order-independent transparency，共支持4种半透明模式：additive，additive with alpha，premultiplied和alpha blend
        * 靠把additive pass单独拆出来，带宽从128bpp优化到80bpp
        * 由于并不是真正的排序，weighted OIT的权重会极大影响最后的混合效果，需要和美术不断沟通调整
    * Hair Rendering：有另一个演讲专门讲Diablo II的头发渲染的，去瞄了眼全是公式和文字，连个图都没有不看了
* A frame on the GPU：通过在Xbox Series X上截帧的结果，分析forward rendering和transparency rendering这两个pass的occupancy都比较低，原因是shader太复杂导致VGPR很高，给了优化VGPR的一些思路
    * 尽量缩短变量的生命周期
    * 尽可能让各个代码块之间相互独立不要有相互依赖
    * 可以尝试重新排序代码逻辑，可能会有奇效
    * QA环节有人提到这些优化可能会在不同平台有不同影响，尽量照顾性能比较弱的平台
* A frame on the CPU：介绍渲染线程的CPU部分，介绍多线程job system，使用RenderGraph架构
* Streaming Performance：简单介绍了streaming的策略
* LODs：使用了Simplygon制作LOD，给出了PS4上不同LODs的指标和性能表现

### [GDC 2022] New Graphics Features for 'Forspoken'

链接：[PDF](https://gdcvault.com/play/1027652/Advanced-Graphics-Summit-New-Graphics)，[Video](https://gdcvault.com/play/1027751/Advanced-Graphics-Summit-New-Graphics)

公司：Luminous Productions Co., Ltd.

简介：作者是开发《最终幻想15》的日本公司，Forspoken是他们开发的一款所谓”AAA“游戏，使用自研引擎Luminous Engine开发，主要介绍引擎渲染的各种features，干货不多随便看看吧：
* Model Rendering：包括GBuffer Layout（Normal使用了20 bits的Octahedral Normal Encoding），使用Translated World Space（就是UE的做法）去做坐标变换来提高矩阵乘法之后的Position精度
* Shadows：使用Static Shadowmap提升CSM性能、Screen Space Shadow、Hybrid Ray-Traced Shadows/Ambient Occlusion
* Lighting & Post Effects：
    * PRT：全局光照使用了PRT，使用ray tracing去bake，先生成cubemaps然后投影到SH上，通过在场景里摆放若干volumes去生成probes，在实时判断这些volumes与视椎体的可见性判断是否需要渲染
    * Specular Correction：由于specular probe的密度远小于irradiance probe，所以会使用Irradiance的采样强度去做Specular Correction，做法是拿irradiance的SH0和ibl的SH0做比值，拿结果去缩放ibl采样的亮度
    * Volumetric Cloud：解决体积云与半透明物体的混合问题，依靠记录若干固定density level的depths值，判断半透明物体位于哪两个levels之间再使用其对应的alpha值与半透明进行混合
    * Wide Gamut：支持广色域空间HDR
* Optimizations：一些优化包括GBuffer Pass如何sort、使用Async Compute提高并行、使用VRS（见GDC 2022的AMD的演讲）减少诸如VFX这种heavy pixel pass的开销
* Automatic LOD Generation：模型自动LOD生成

### [GDC 2022] Breaking Down the World of Athia: The Technologies of Forspoken

链接：[Video](https://gdcvault.com/play/1027809/)

公司：AMD, Luminous Productions

简介：主要是为了推销AMD的技术，介绍AMD帮助Forspoken做的一些features和优化，包括：
* Single Pass Downsampler：一个pass输出所有的dowmsampled mips
* CACAO：Ambient Occlusion，可以Async并行加速
* RTAO：离摄像机一定距离以内的使用ray traced ao，以外用CACAO，使用ADM Shadow Denoiser
* Variable Shading：VRS，通过上一帧每个tile的luminance variation来决定这帧的shading rate，需要AMD硬件支持
* Hybrid Shadows：判断当前位置是否需要tracing，还是使用shadowmap就足够了
* FidelityFX Super Resolution：类似DLSS
* Microsoft DirectStorage：加快加载速度

### [GDC 2022] A Guided Tour of Blackreef: Rendering Technologies in Deathloop

链接：[Video](https://gdcvault.com/play/1028106/)

公司：AMD

### [GDC 2022] Simulating Tropical Weather in 'Far Cry 6'

链接：[PDF](https://gdcvault.com/play/1027675/Simulating-Tropical-Weather-in-Far)，[Video](https://gdcvault.com/play/1027725/Simulating-Tropical-Weather-in-Far)

公司：Ubisoft Toronto, Ubisoft Montreal

简介：介绍Far Cry 6里独特的热带天气系统是如何实现的：
* Core Controls：介绍了Weather Manager的使用，游戏分为3种Region，每种region会以5天为单位循环天气，靠文本文件定义了一天内每个小时的天气preset
* Material Wetness：介绍如何实现材质的Wetness效果，主要分为Static和Dynamic两种方式
    * Static用于静态物体，会渲染一张Wetness Shadowmap来避免在室内等区域错误地改变了场景湿度，同时每个材质控制Porosity属性来决定它受Wetness的影响程度，在Deferred Lighting时根据Porosity来修改albedo、smoothness、metallic和specular等材质属性
    * Dynamic用于角色、武器和载具等动态物体，需要额外的raycast来判断本身是否需要暴露在雨中，并依靠单独的Shader逻辑来应用Wetness，所以更加自由可控。介绍了对角色、武器、载具和植被的Dynamic Wetness的特殊处理，例如怎么渲染雨滴等效果
    * 如何处理Terrain的下雨效果
* Rendering Features：介绍了大量跟天气系统相关的渲染features是如何实现的
    * Lighting：简单介绍使用的BRDF
    * Global Illumination：基于美术手摆的Probe，烘焙了11个时间点的光照+Sky Occlusion+Local Lights=13个frames的probes lighting数据
    * Atmospheric Scattering、Volumetric Cloud、Volumetric Fog：都是比较常规的方法，体积云是半分辨率渲染，用了Checkboard做upsampling。体积雾为了减弱shafts因为低分辨率导致的阶梯状瑕疵使用了Exponential Blurred Shadows
    * Reflection：有另一个专门的演讲介绍Far Cry 6的Raytracing方案计算反射，这里主要就介绍了作为fallback的Cubemap方案。美术手摆Probe，每个存一套GBuffer然后每帧计算Relighting，不考虑shadow，每帧只relight一个face
    * Rain：GPU粒子的方案
    * Lightning、Ocean、Tree Bending

### [Digital Dragons 2021] Rendering Watch Dogs

链接：[Video](https://www.youtube.com/watch?v=RUWAVErMJto)

公司：Ubisoft

## Lighting

光照综合向相关。

### [GDC 2022] Advanced Graphics Summit: 'Cyberpunk 2077': Bringing Light to Night City

链接：[Video](https://gdcvault.com/play/1027959/Advanced-Graphics-Summit-Cyberpunk-2077)

公司：CD Projekt RED

简介：介绍了2077在Lighting方面所做的工作，包括：
* Visual idea：视觉设定
    * 介绍如何营造赛博朋克风格分光影，参考了攻壳机动队、星际穿越等电影作为参考
    * 制作Lookdev，编写了Nuke脚本生成Tonemapping使用的LUT，使得16位的Lighting结果经过LUT映射后的画面更加具有电影感，并且为SDR和HDR分别生成了两种LUT
* Setting the Stage：
    * 24-hour Cycle：24小时的光照参数在物理光照参数的基础上进行了一定缩放，来得到更好的曝光和视觉效果，比如拉长了magic hour/blur hour时间、增强了月光和夜晚天空的环境光强度、大大降低了太阳光/天光的强度（大约100倍）等
    * Global Illumination：GI方案在参考了全境封锁的Probe + Surfel方案的基础上做了一些改善，包括使用Volume控制Probe密度、增加了防漏光数据等等，可以覆盖摄像机周围128~192米左右，超过这个距离就会fallback到Distance GI（使用topdown的RSM，竖直方向靠Reflection Probe的Visibility去弥补）
    * Reflection Probes：使用层级摆放的Reflection Probe，Global Probes -> Large City Probes -> Detailed City Probe -> Interior Probes，依次精度和优先级更高，最后SSR会作为最高级别的反射来源
* Ray Tracing：项目后期NVIDIA团队帮忙支持Ray Tracing，使得GI可以使用更加统一方法实现
    * 使用了Hybrid Raytraced的方式来进一步提升GI效果解决复杂结构的全局光照问题，具体方式是用Raytracing的结果来替代GI系统中的Sky Light和Sun Light的第一次反弹贡献，其他部分保持不变
    * 使用Ray Tracing计算自发光物体光源、RTAO和Reflections
    * 增加了Raytraced Reference模式来使用光追完全替代全局光照的计算，不依赖任何传统的Subsystem，这一功能计划未来开放给玩家使用
* Lights：
    * Capsule Light：面光的实时开销太大了，所以使用Capsule Light来模拟各种面光源，结合Volumetric Fog营造氛围
    * Dynamic GI with Portal Light：动态GI方案由于精度问题可能会导致室内光照比室外暗很多，并且由于性能问题GI并不会参与贡献Specular Response。为了给GI补光，增加了跟着TOD变化的Portal Lights，这些光源通常被放置在窗户和门口附近，美术也可以进一步调整光源参数来定制化效果
    * Character Lighting：角色光照单独打光，使用Raytraced Shadow得到更加正确柔和的阴影效果
* Managing the Budget：解决性能问题
    * Large Angle/Radius Lights：把shadow radius和light radius分离开来调整，这样即便light radius很大，它的投影范围也不用那么大来节省性能，同时也能提高shadowmap利用率
    * High Light Density：使用Fog-only Lights来尽可能减少真正产生光照的光源数目；使用Light Channel Volume来指定光源可以影响到的空间区域，尽可能减少需要投影的光源；使用streaming distance和streaming range控制光源投影的淡入淡出距离；Dynamic Shadow Lights最多6个，Static Shadow Lights最多4个，后者的shadow不需要每帧更新
    * Deep Vistas：把光源bake到Distance Lights，只计算非常简化的diffuse only光照结果，会考虑光源的inner和outer angle以及模拟真实光源的闪烁，在渲染时会考虑Occlusion Query来判断是否需要渲染，在超远视距下和真正的光照结果做淡入淡出；使用Light Pollution Particles来减弱超远视距下的环境视觉噪声，和真正的Volumetric Fog效果做淡入淡出

### [GDC 2022] Recalibrating Our Limits: Lighting on 'Ratchet and Clank: Rift Apart'

链接：[PDF](https://gdcvault.com/play/1027666/Recalibrating-Our-Limits-Lighting-on)，[Video](https://gdcvault.com/play/1027792/Recalibrating-Our-Limits-Lighting-on)

公司：Insomniac Games

简介：制作蜘蛛侠、瑞奇与叮当系列的公司，比较轻松的演讲。印象最深刻的是大家都是远程工作的，达到这样的品质很不容易：
* Pre-production：介绍新作打光的pipeline
    * Establish Base Exposure：先确定曝光值，这几乎会影响所有的东西，但曝光合适与否又不容易看出来，尤其在项目前期如果定了一个错误的曝光可能一时半会看不出来，到了后面才逐渐发现很麻烦，总体来说这些影响集中在自发光材质和特效、灯光强度、自动曝光范围
    * Establish Key Lights：确定主光源强度、设置直接光和间接光比值（大约3:1），其他光源的亮度可以据此来调整
    * Place Lights：如何用尽可能少但范围大的光源去有效打光
    * Adjust Lights：修改光照、重新烘焙、检查结果，重复polish直到得到满意的效果
    * Post and Atmosphere：调整后处理和雾效等
* An Example：给了一个无效打光的场景示例，虽然场景里打了很多光但看上去就是不好看，给出了怎么改善的过程
* Performance：简单介绍了一些优化策略，包括如何减少角色光照的开销（新开发的角色毛发效果增加了很多开销，解决方法是让fur shells不参与shadow pass，这样一来可以减少开销二来有助于得到假的毛发SSS效果，后续在deferred lighting的时候使用screen space shadow来给毛发增加阴影效果）、优化Raytracing效率、增加LOD等
* Developing A Look：简单介绍了下怎么从原画到最终效果验收合格的一个场景示例
* QA环节有人问如何保证SDR和HDR效果一致性，回答是美术都是在HDR显示器下工作的（壕），会经常跑游戏看效果

### [GDC 2021] Advanced Graphics Summit: Lifting the Fog: Geometry & Lighting in 'Demon's Souls'

链接：[PDF](https://gdcvault.com/play/1027011/Advanced-Graphics-Summit-Lifting-the), [Video](https://gdcvault.com/play/1027469/Advanced-Graphics-Summit-Lifting-the)

公司：Bluepoint Games

### [SIGGRAPH 2021] Real-Time Samurai Cinema: Lighting, Atmosphere, and Tonemapping in Ghost of Tsushima

链接：[PDF](http://advances.realtimerendering.com/s2021/jpatry_advances2021/index.html), [Video](https://www.youtube.com/watch?v=GOee6lcEbWg&feature=emb_title)

公司：Sucker Punch Productions

## Global Illumination

全局光照相关。

### [SIGGRAPH 2021] Large-Scale Global Illumination at Activision

链接：[Video](https://www.youtube.com/watch?v=snXTGrjfOvQ)

公司：Activision

### [SIGGRAPH 2021] Radiance Caching for Real-Time Global Illumination

链接：[Video](https://www.youtube.com/watch?v=2GYXuM10riw&feature=emb_title)

公司：Epic Games

### [SIGGRAPH 2021] Global Illumination Based on Surfels

链接：[Video](https://www.youtube.com/watch?v=Uea9Wq1XdA4&feature=emb_title)

公司：Electronic Art

## Shadows

阴影相关。

### [GDC 2021] Advanced Graphics Summit: Shadows of Cold War: A Scalable Approach to Shadowing

链接：[Video](https://gdcvault.com/play/1027442/Advanced-Graphics-Summit-Shadows-of)

公司：Treyarch

## Volumetrics

体渲染相关。

### [GDC 2022] The Real-Time Volumetric Superstorms of 'Horizon Forbidden West'

链接：[PDF](https://gdcvault.com/play/1027688/The-Real-Time-Volumetric-Superstorms)，[Video](https://gdcvault.com/play/1028023/The-Real-Time-Volumetric-Superstorms)

公司：Guerrilla Games

简介：作者之前是在Blue Sky动画公司做体积云模拟和渲染的，后来在2013年加入了Guerrilla Games，这次主要分享怎么扩展整个Cloud System去支持暴风雨效果的。前半部分主要回顾之前NUBIS系统是如何渲染体积云的，基本在之前的演讲里都介绍过了。下半部分介绍如何对系统做扩展去支持渲染暴风雨效果，可参考A站上给的效果https://www.artstation.com/delta307，演讲包括：
* Modeling Density：对暴风雨的云密度进行建模来模拟云的形状，主要包括两种云：
    * Anvil Cloud：砧积云，即顶部的云团
    * Mesocyclone：中气旋，即暴风雨中心的旋状云团，也是龙卷风形成的地方
* Modeling Light：模拟特有的光照表现，基本是靠各种probability field去trick各种变化效果，包括：
    * Attenuation和Ambient：模拟云的光照衰减和环境光效果
    * Red Glow：避免计算二次raymarching，使用一个球状光源的probability来模拟
* Lightning Effects：模拟闪电
* Environmental Effects：与环境表现结合
* Controlling Chaos：整个世界分为5个可以产生暴风雨的区域，每个区域有自己特有的配置。每分钟会随机从其中挑选一个位置，然后在2分钟左右的时间加载暴风雨，持续一段时间后再在2分钟消散，持续上述循环直到玩家完成修复天气控制系统的任务
* Scaling PS4 & PS5：给出了PS4和PS5上的一些渲染配置以及PS4的优化，全看天空的时候控制在4ms以内，其他时间控制在2ms左右
* Extension：正在开发新的功能让玩家可以自由在云里穿梭（参见：https://www.artstation.com/artwork/ZeXyPZ）

## Ray Tracing

光线追踪相关。

### [GDC 2022] Performant Reflective Beauty: Hybrid Raytracing with Far Cry 6

链接：[Video](https://gdcvault.com/play/1028107/)

公司：Ubisoft Toronto, AMD

简介：介绍Far Cry 6是热带天气风格的FPS游戏，经常下雨导致场景有大量反射，使用了SSR和硬件Raytracing结合的方案，过程如下：
* Generate Rays：根据GGX BRDF得到反射lobe，把生成的rays存储到半分辨率的buffer里
* SSLR：Screen Space Local Reflections，比Raytracing要快，但有一些瑕疵：
    * 在有限步长下结果不稳定：解决方法是使用Linear Trace With Refine，即不使用HiZ而是使用Linear Trace，最多64个large steps后跟8个refine steps，性能跟HiZ一样但更加稳定），为了防止large step会略过一些edges，使用random ray start offsets
    * SS  Tile Classifications：优化策略，光滑表面需要的精度更高而粗糙表面精度不需要很高，所以可以根据roughness计算出来的cone angle包含的像素数选择depth mip
* Ray Trace & Lighting：远距离用SS，近距离用HW RT，中间的区域靠计算confidence决定混合
* Particles Trace：如何在HW RT中计算2D billboards，由于billboard始终面向摄像机所以默认下在ray trace的时候可能刚好旋转角度不可见，解决方法是使用3个相互交叉的片去模拟粒子，在trace的时候选择与射线垂直的那个平面计算简化后的光照

### [GDC 2022] Real-Time Ray Tracing in 'Hitman 3'

链接：[Video](https://gdcvault.com/play/1027665/Advanced-Graphics-Summit-Real-Time)

公司：IO Interactive

### [GDC 2022] Bringing 4K Ray Traced Visuals to the World of Hitman 3

链接：[Video](https://gdcvault.com/play/1027828/)

公司：IO Interactive, Intel

## Terrain & Grass

地形和植被系统相关。

### [GDC 2022] Adventures with Deferred Texturing in 'Horizon Forbidden West'

链接：[PDF](https://gdcvault.com/play/1027553/Adventures-with-Deferred-Texturing-in)，[Video](https://gdcvault.com/play/1028035/Adventures-with-Deferred-Texturing-in)

公司：Guerrilla

### [GDC 2022] Advanced Graphics Summit: Designing the Terrain System of 'Flight Simulator': Representing the Earth

链接：[PDF](https://gdcvault.com/play/1027581/Advanced-Graphics-Summit-Designing-the)，[Video](https://gdcvault.com/play/1027838/Advanced-Graphics-Summit-Designing-the)

公司：Asobo Studio

简介：法国游戏工作室，之前制作过《瘟疫传说》等游戏，这次介绍微软飞行模拟器是如何渲染地形相关的各种features的，前两个部分暂时不感兴趣先略过：
* Terrain System Architecture：基于四叉树
* Flexibility：保证系统足够灵活
* Scalability：
    * Large Coordinates：CPU使用double 64位，GPU使用Inverted Depth Buffer和Anchor Space（原点是摄像机，竖直Y轴是当前位置法线方向）
    * Trees：要渲染上百万个树，使用了堡垒之夜的3D Imposter的方法
    * Shadows：和[大表哥2](http://candycat1992.github.io/2021/12/12/rdr2-shadows/)使用了类似的阴影方案，CSM + Terrain Shadow (Topdown渲染地形到heightmap上，然后raymarch这张heightmap) + Small Shadows (Screen Space Raymarched Shadows)

### [GDC 2021] Advanced Graphics Summit: Procedural Grass in 'Ghost of Tsushima'

链接：[PDF](https://gdcvault.com/play/1027033/Advanced-Graphics-Summit-Procedural-Grass)

公司：Sucker Punch Productions

### [GDC 2021] Samurai Landscapes: Building and Rendering Tsushima Island on PS4

链接：[Video](https://gdcvault.com/play/1027352/Samurai-Landscapes-Building-and-Rendering)

公司：Sucker Punch Productions

### [GDC 2021] Boots on the Ground: The Terrain of Call of Duty

链接：[PDF](https://research.activision.com/publications/2021/09/boots-on-the-ground--the-terrain-of-call-of-duty)

公司：Teryarch

### [SIGGRAPH 2021] Experimenting with Concurrent Binary Trees for Large Scale Terrain Rendering

链接：[Video](https://www.youtube.com/watch?v=0TzgFwDmbGg)

公司：Unity Technologies

## Wind Simulation

风场模拟相关。

### [GDC 2021] Blowing from the West: Simulating Wind in 'Ghost of Tsushima'

链接：[PDF](https://gdcvault.com/play/1027124/Blowing-from-the-West-Simulating)，[Video](https://blog.playstation.com/2021/01/12/how-stunning-visual-effects-bring-ghost-of-tsushima-to-life/		)							
公司：Sucker Punch Productions

## Particles

粒子相关。

### [GDC 2021] Enhancement of Particle Simulation Using Screen Space Techniques in 'The Last of Us: Part II'

链接：[Video](https://gdcvault.com/play/1027356/Enhancement-of-Particle-Simulation-Using)

公司：Naughty Dog

## Open World

大世界相关。

### [GDC 2021] Zen of Streaming: Building and Loading 'Ghost of Tsushima'

链接：[Video](https://gdcvault.com/play/1027205/Zen-of-Streaming-Building-and)

公司：Sucker Punch Productions

## GPU Driven

大量依赖GPU算力的都放这里了。

### [SIGGRAPH 2021] A Deep Dive into Nanite Virtualized Geometry

链接：[Video](https://www.youtube.com/watch?v=eviSykqSUUw&feature=emb_title)

公司：Epic Games

## Post Process

后处理相关。

### [GDC 2022] FidelityFX Super Resolution 2.0

链接：[Video](https://gdcvault.com/play/1027829/)

公司：AMD

### [SIGGRAPH 2021] Improved Spatial Upscaling through FidelityFX Super Resolution for Real-Time Game Engines

链接：[Video](https://www.youtube.com/watch?v=aKyjQPq5aUU)

公司：AMD

## Hair & Fur

毛发渲染相关。

### [GDC 2022] Hair and Fur Rendering in 'Diablo II: Rescurrected'

链接：[Video](https://gdcvault.com/play/1028080/Hair-and-Fur-Rendering-in)

公司：Bizzard Entertainment

简介：感觉作者没好好准备，约等于一张演示图都没有，目前看不下去。

## Procedural

过程化生成相关。

### [GDC 2022] Never The Same Twice: Procedural World Handling in 'Returnal'

链接：[Video](https://gdcvault.com/play/1028093/Never-The-Same-Twice-Procedural)

公司：Housemarque

## Visual Arts

TA管线或经验相关。

### [GDC 2022] Visual Effects Summit: How to (Not) Create Textures for VFX

链接：[Video](https://gdcvault.com/play/1027741/Visual-Effects-Summit-How-to)

公司：Wild Sheep Studio

### [GDC 2022] Technical Artist Summit: Lighting the Way: Efficient Dynamic Lighting & Shadows on Mobile

链接：[Video](https://gdcvault.com/play/1027858/Technical-Artist-Summit-Lighting-the)

公司：NetEase Games

简介：介绍网易动作手游《Dark Bind》的渲染技术，非常水随便看看：
* Point Light的阴影渲染使用了Dual Paraboloid Shadow Mapping，缺点是阴影会有所变形
* Bloom使用Dual Filtering替代Gaussian Blur，从1.1ms可以优化到0.5ms

### [GDC 2022] Technical Artist Summit: Finding Harmony in Anime Style and Physically Based Rendering

链接：[PDF](https://gdcvault.com/play/1027597/Technical-Artist-Summit-Finding-Harmony)，[Video](https://gdcvault.com/play/1027995/Technical-Artist-Summit-Finding-Harmony)

公司：NetEase Games

简介：介绍网易自研Messiah引擎制作的二次元游戏Mirage的卡通渲染中的关键技术，游戏风格是和PBR结合的NPR渲染，支持local lights等光照结果，基于Hybrid Forward + Deferred Lighting，大部分技术都见过了，有些有点意思的点：
* Face Lighting：由于脸部的Flat Shading导致多光源叠加的时候能量不守恒脸部亮度过曝，所以对non shadow shadow做了wrap处理，对main light来说会计算statistical coverage precomputation
* Skin Lighting：根据直接光的luminance去shift hue，提高暗部和阴影交界线的饱和度
* Shadow：
    * Hair使用Shadow Proxy Mesh去绘制阴影、GI和AO等
    * 脸部为了防止接受到环境杂乱的阴影，用了Partial Shadow（没太理解）
* Hair Lighting：各向异性高光不想在多光源下过于杂乱，所以没有使用Kajiya-Kay的half vector去计算，而是从view direction和main light direction插值得到的一个方向，把它计算得到的specular mask直接存到GBuffer里给所有local lights使用

### [GDC 2022] Technical Artist Summit: Bringing the World to Your Shaders

链接：[PDF](https://gdcvault.com/play/1027568/Technical-Artist-Summit-Bringing-the)，[Video](https://gdcvault.com/play/1028009/Technical-Artist-Summit-Bringing-the)

公司：Epic Games

### [GDC 2022] Building the World of 'The Ascent'

链接：[Video](https://gdcvault.com/play/1027800/Building-the-World-of-The)

公司：Neon Giant

简介：介绍了《The Ascent》的视觉效果是如何实现的。The Ascent是用UE4制作的一款RPG游戏，这次分享介绍了小团队（位于瑞典，目前团队11人）是如何非常快速有效地制作这样一款看起来细节很丰富的游戏：
* Modeling Workflow：为了能够最快速地迭代场景制作，整个游戏只有低模没有使用任何高模，所有资产几乎只用了同一张Texture（包括了trimsheets和用于表现细节的纹理结构），大量使用decals来隐藏纹理的重复性
* Shader Worldflow：最有亮点的部分，由于全场景只使用一张Texture，需要使用一种快速给物体上色制作细节的方法，无法忍受给每个asset单独制作很多个材质人力成本太高，绘制mask也不行。解决方法核心是靠类似UDIM的思想靠不同UV空间去控制颜色和各种属性，在此基础上叠加各个layer（dirt、emissive layers等）来得到科幻废土风格的效果。好处是纹理内存非常少（因为只有一张纹理），制作流程和规则简单明了强制美术风格的一致，技术非常可控，场景中的顶点复杂度、材质复杂度等都完全在预知范围内，非常有利于做优化
* Assembing the World：介绍了场景制作时使用的自动化工具，主要是基于Houdini的PCG工具，包括管道、广告牌和破碎效果等，比较常规
* Adding Life to the World：介绍如何让场景丰富生动，靠AI驱动NPC，以及模拟大量广告牌灯光（在Houdini里处理所有的广告片段把每一帧blur到一个pixel，导出这些颜色序列帧成CSV文件并导入到UE4，实时读取这些颜色去驱动灯光动画，伪造广告牌光照效果）等
* Optimization：介绍一些灯光优化，主要是靠调整UE4的参数实现，比如调大lightmap分辨率（因为其他纹理内存占用非常少）来减少dc，调整volumetric lightmap密度减少内存，以及调小Reflection Probe的分辨率。游戏里大量使用手绘的Reflection Probe，即反射内容和场景不完全一致而是由美术决定，因为游戏视角比较固定不会像FPS游戏那样所以可行
* Warm Up：最后反思了一些经验教训，制定规则虽然很烦人但可以强制大家关注big picture，同时由于本作太关注制作速度和数量，有些忽略了质量，希望在下一作做得更好

### [GDC 2022] Creating the Many Faces of 'Horizon Forbidden West'

链接：[Video](https://gdcvault.com/play/1027878/Creating-the-Many-Faces-of)

公司：Guerilla Games

简介：介绍地平线2是如何在三年时间内制作了168个高质量Faces的：
* Creating Our Heros：扫描和制作Hero的人脸模型，这里是挑选特定长相的模特，后面会再基于扫描数据做一些风格化处理
* Populating the Trbes：这一步目的是既要保证角色的多样性，又要保证质量可以和Heroes相当。扫描各种真实的人脸模型，组成GenePool数据库，这里只考虑数据多样性（形状特征明显，各种不同年龄、种族），而不会想着某种特定的样子。GenePool包含60个模型数据，具有完整的Rigging数据，而美术在ZBrush制作游戏真正的角色模型缺少Rigging，此时就会使用内部工具GeneSplicer，它会识别美术制作的模型，识别出它的各个部分与数据库里匹配的数据，在GenePool的大量数据样本之间做Blend，生成具有完整Rigging信息的人脸模型
* Look Development：在游戏引擎内部制作Lookdev环境，介绍了脸部绒毛（一开始靠插片但overdraw太高了所以改为插很多更高精度的polygons）、肤色（Albedo本身是灰色调的图，靠另一张查找表给皮肤上色）等角色制作方法
* Living in the World：靠decals、face paint等方法进一步丰富脸部表现

### [GDC 2021] The Art of Not Reinventing the Wheel in 'Wild Rift' Asset Pipeline

链接：[PDF](https://gdcvault.com/play/1027061/Technical-Artist-Summit-The-Art)

公司：Riot Games

## 杂七杂八

不知道分类到哪里。

### [GDC 2022] Shifts and Rifts: Dimensional Tech in 'Ratchet and Clank: Rift Apart'

链接：[Video](https://gdcvault.com/play/1027872/Shifts-and-Rifts-Dimensional-Tech)

公司：Insomniac Games

简介：介绍如何在瑞奇与叮当中实现高效的传送门效果：
* Portal Rendering：介绍了Portal的渲染实现（Render to Texture）以及遇到的各种问题和解决方法，包括：
    * Lighting：解决两个世界光强差异过大的问题，做一次反向Tonemapping或者直接关闭Tonemapping，如果光强仍然不一样会再使用一个自定义光强去调整Portal
    * Motion Blur：解决Portal显示的诸如Motion Blur和Depth of Field等后处理错误问题，方法是把Portal的Depth和Velocity Buffer等拷贝到Main Buffer里
    * Camera Popping：解决角色越过Portal时摄像机突变，方法是在玩家跨过Portal时不要立刻切换摄像机，而是继续追踪一个虚拟的玩家目标，直到整个摄像机也一起跨过Portal后再重设成真正的玩家目标
    * Portal Threshold：解决角色在跨过边界的时候会被错误裁剪的问题，方法是在Portal两侧复制两个相同的物体，两边的角色模型分别在Shader里使用了Clipping来避免绘制到Portal模型的另一侧
    * Inputs：解决进入Portal后角色移动与玩家输入不匹配的问题，原因是输入移动方向是基于入口处的观察空间的，尽管模型移动轨迹在此空间下的确是正确的，但因为玩家是从Portal处观察移动方向的所以看起来是错误的。方法就是对输入方向做一次转换，当模型跨过Portal后，使用输入方向和Portal的相对方向去控制玩家移动方向
    * Optimizations：优化Portal的渲染效率，包括使用Portal Model作为遮挡等
* Minimizing Load Times：介绍如何解决传送门在两个世界传送时快速Loading，最后成功让Loading时间在PS5上控制在1秒以内

### [GDC 2022] 'Dead by Daylight': Intergrating Shaders into the User Interface Pipeline

链接：[Video](https://gdcvault.com/play/1027893/-Dead-by-Daylight-Integrating)

公司：Epic Games

### [GDC 2021] Driving Innovation: A New Vehicle Pipeline for 'The Last of Us: Part II'

链接：[PDF](https://gdcvault.com/play/1027001/Driving-Innovation-A-New-Vehicle)

公司：Naughty Dog

### [GDC 2021] Rope Simulation in 'Uncharted 4' and 'The Last of Us: Part 2'

链接：[Video](https://gdcvault.com/play/1027351/Rope-Simulation-in-Uncharted-4)

公司：Naughty Dog