Unreal Engine - Contact Shadow
=============
### 3.23
 
阅读 UE 对 Contact Shadow 实现源码


```c++
ScreenSpaceShadowRayCast.ush // 核心算法实现

// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "Common.ush"
#include "SceneTexturesCommon.ush"

// Returns distance along ray that the first hit occurred, or negative on miss
float CastScreenSpaceShadowRay(
	float3 RayOriginTranslatedWorld, float3 RayDirection, float RayLength, int NumSteps,
	float Dither, float CompareToleranceScale, bool bHairNoShadowLight,
	out float2 HitUV)
{
	const float4 RayStartClip = mul(float4(RayOriginTranslatedWorld, 1), View.TranslatedWorldToClip);
	const float4 RayDirClip = mul(float4(RayDirection * RayLength, 0), View.TranslatedWorldToClip);
	const float4 RayEndClip = RayStartClip + RayDirClip;

	const float3 RayStart = RayStartClip.xyz / RayStartClip.w;
	const float3 RayEnd = RayEndClip.xyz / RayEndClip.w;

	const float3 RayStep = RayEnd - RayStart;

	// 假设所有像素背后的物体，都至少有 RayLength 那么厚，这里只是估计，利用已有信息算不出精确的物体厚度
	// 该厚度用于后续判断光线是否穿出物体后面，避免近处的物体遮挡远处的物体
	const float4 RayDepthClip = RayStartClip + mul(float4(0, 0, RayLength, 0), View.ViewToClip);
	const float3 RayDepth = RayDepthClip.xyz / RayDepthClip.w;

	const float StepOffset = Dither - 0.5f;
	const float Step = 1.0 / NumSteps;

	const float CompareTolerance = abs(RayDepth.z - RayStart.z) * Step * CompareToleranceScale;

	float SampleTime = StepOffset * Step + Step;

	const float StartDepth = LookupDeviceZ(RayStart.xy * View.ScreenPositionScaleBias.xy + View.ScreenPositionScaleBias.wz);

	UNROLL
	for (int i = 0; i < NumSteps; i++)
	{
		// SamplePos is in NDC space of the current view
		const float3 SamplePos = RayStart + RayStep * SampleTime;
		const float2 SampleUV = SamplePos.xy * View.ScreenPositionScaleBias.xy + View.ScreenPositionScaleBias.wz;
		const float SampleDepth = LookupDeviceZ(SampleUV);

		// Avoid self-intersection with the start pixel (exact comparison due to point sampling depth buffer)
		// Exception is made for hair for occluding transmitted light with non-shadow casting light
		// 这里主要是为了避免在起始像素发生自相交，因为采样的是同一张深度图，所以如果深度值相同，说明是同一个像素，直接跳过就行了
		if (SampleDepth != StartDepth || bHairNoShadowLight)
		{
			// 对于射线飞到了近处的物体后面，DepthDiff会变成很大的负数，这时就不应该算作命中，避免近处的物体遮挡了远处的物体
			const float DepthDiff = SamplePos.z - SampleDepth;
			const bool bHit = abs(DepthDiff + CompareTolerance) < CompareTolerance;

			if (bHit)
			{
				HitUV = SampleUV;

				// Off screen masking, check NDC position against NDC boundary [-1,1]
				bool bValidPos = all(and(-1.0 < SamplePos.xy, SamplePos.xy < 1.0));

				return bValidPos ? (RayLength * SampleTime) : -1.0;
			}
		}

		SampleTime += Step;
	}

	return -1;
}
```

有两个调用处，一个在 DeferredLightingCommon.ush，另一个在 ScreenSpaceShadow.usf 关闭 Bend_SSS 宏的时候，用的时候记得只开一边就好

```c++
DeferredLightingCommon.ush // 实际调用处

void ApplyContactShadowWithShadowTerms(
	float SceneDepth, 
	uint ShadingModelID, 
	float ContactShadowOpacity, 
	FDeferredLightData LightData, 
	float3 TranslatedWorldPosition, 
	half3 L, 
	float Dither, 
	inout FShadowTerms OutShadow)
{

    ...

    BRANCH
    if (ContactShadowLength > 0.0)
    {
        bool bHitCastContactShadow = false;
        bool bHairNoShadowLight = ShadingModelID == SHADINGMODELID_HAIR && !LightData.ShadowedBits;
        // 在这里调用 CastScreenSpaceShadowRay
        float HitDistance = ShadowRayCast( TranslatedWorldPosition, L, ContactShadowLength, 8, Dither, bHairNoShadowLight, bHitCastContactShadow );
        
        // 实际应用接触阴影距离的地方
        if ( HitDistance > 0.0 )
        {
            float ContactShadowOcclusion = bHitCastContactShadow ? LightData.ContactShadowCastingIntensity : LightData.ContactShadowNonCastingIntensity;

            // Exponential attenuation is not applied on hair/eye/SSS-profile here, as the hit distance (shading-point to blocker) is different from the estimated 
            // thickness (closest-point-from-light to shading-point), and this creates light leaks. Instead we consider first hit as a blocker (old behavior)
            BRANCH
            if (ContactShadowOcclusion > 0.0 && 
                IsSubsurfaceModel(ShadingModelID) &&
                ShadingModelID != SHADINGMODELID_HAIR &&
                ShadingModelID != SHADINGMODELID_EYE &&
                ShadingModelID != SHADINGMODELID_SUBSURFACE_PROFILE)
            {
                // Reduce the intensity of the shadow similar to the subsurface approximation used by the shadow maps path
                // Note that this is imperfect as we don't really have the "nearest occluder to the light", but this should at least
                // ensure that we don't darken-out the subsurface term with the contact shadows
                float Density = SubsurfaceDensityFromOpacity(ContactShadowOpacity);
                ContactShadowOcclusion *= 1.0 - saturate( exp( -Density * HitDistance ) );
            }

            float ContactShadow = 1.0 - ContactShadowOcclusion;

            OutShadow.SurfaceShadow *= ContactShadow;
            OutShadow.TransmissionShadow *= ContactShadow;
        }
    }

    ...

}
```

```c++
ScreenSpaceShadow.usf // 实际调用处

[numthreads(THREADGROUP_SIZEX, THREADGROUP_SIZEY, 1)]
void ScreenSpaceShadowsCS(
uint3 GroupId : SV_GroupID,
uint3 DispatchThreadId : SV_DispatchThreadID,
uint3 GroupThreadId : SV_GroupThreadID)
{
uint ThreadIndex = GroupThreadId.y * THREADGROUP_SIZEX + GroupThreadId.x;

	float2 ScreenUV = (DispatchThreadId.xy * DownsampleFactor + ScissorRectMinAndSize.xy + .5f) * View.BufferSizeAndInvSize.zw;
	float2 ScreenPosition = (ScreenUV - View.ScreenPositionScaleBias.wz) / View.ScreenPositionScaleBias.xy;

	float SceneDepth = CalcSceneDepth(ScreenUV);
	float3 OpaqueTranslatedWorldPosition = mul(float4(GetScreenPositionForProjectionType(ScreenPosition, SceneDepth), SceneDepth, 1), PrimaryView.ScreenToTranslatedWorld).xyz;

	const float ContactShadowLengthScreenScale = GetScreenRayLengthMultiplierForProjectionType(SceneDepth).y;
	const float ActualContactShadowLength = ContactShadowLength * (bContactShadowLengthInWS ? 1.0f : ContactShadowLengthScreenScale);

	const float Dither = InterleavedGradientNoise(DispatchThreadId.xy + 0.5f, View.StateFrameIndexMod8);

	// 实际应用接触阴影距离的地方
	float2 HitUV;
	float HitDistance = CastScreenSpaceShadowRay(OpaqueTranslatedWorldPosition, LightDirection, ActualContactShadowLength, 8, Dither, 2.0f, false, HitUV);

	float Result = HitDistance > 0.0 ? (1.0f - GetContactShadowOcclusion(HitUV)) : 1.0f;

	#if METAL_ES3_1_PROFILE 
		// clamp max depth to avoid #inf
		SceneDepth = min(SceneDepth, 65500.0f);
	#endif
	RWShadowFactors[DispatchThreadId.xy] = float2(Result, SceneDepth);
}
```

### 3.26

搭建场景并对 Contact Shadow 做基础测试

<div style="display: flex; justify-content: center; gap: 10px;">
  <div style="text-align: center;">
    <img src="./blog_img/Cascade.png" style="width: 100%;" />
    <p>Cascade</p>
  </div>
  <div style="text-align: center;">
    <img src="./blog_img/Cascade+ContactShadow.png" style="width: 100%;" />
    <p>Cascade + ContactShadow</p>
  </div>
</div>

参考 https://winter-crown-works.com/en/tech/003/#before-depthoffset-depthbuffer 接入 BendSSS

<div style="display: flex; justify-content: center; gap: 10px;">
  <div style="text-align: center;">
    <img src="./blog_img/Cascade+ContactShadow.png" style="width: 100%;" />
    <p>Cascade + ContactShadow</p>
  </div>
  <div style="text-align: center;">
    <img src="./blog_img/Cascade+DeferredLight+Bend.png" style="width: 100%;" />
    <p>Cascade + DeferredLight + Bend</p>
  </div>
</div>

### 3.29

查看 RenderScreenSpaceShadows 的调用点

1. FDeferredShadingSceneRenderer::RenderDeferredShadowProjections，这里是我手动添加的调用点。调用链路：FDeferredShadingSceneRenderer::Render -> FDeferredShadingSceneRenderer::RenderLights -> FDeferredShadingSceneRenderer::RenderDeferredShadowProjections -> RenderScreenSpaceShadows

2. FMobileSceneRenderer::RenderMobileShadowProjections，这里是 UE 移动端的调用点。调用链路：FMobileSceneRenderer::Render -> FMobileSceneRenderer::RenderMobileShadowProjections -> RenderScreenSpaceShadows

注意在这边开启之后，LightRendering.cpp 那边就要关掉 Contact Shadow 的渲染，不然会重复渲染两遍 Contact Shadow，具体做法是把 ApplyContactShadowWithShadowTerms() 的两个调用点全部注释掉

<div style="display: flex; justify-content: center; gap: 10px;">
  <div style="text-align: center;">
    <img src="./blog_img/Cascade+DeferredLight+Bend.png" style="width: 100%;" />
    <p>Cascade + DeferredLight + Bend</p>
  </div>
  <div style="text-align: center;">
    <img src="./blog_img/Cascade+Pure_Bend.png" style="width: 100%;" />
    <p>Cascade+PureBend</p>
  </div>
</div>

### 3.30

RenderScreenSpaceShadowsBend 的具体实现

首先 CPU 侧调用原博客中提供的 bend_sss_cpu.h 进行数据准备，再推送给 GPU 调用到 ScreenSpaceShadow.usf 进行实际的计算。

在 ScreenSpaceShadow.usf 中，如果在 ScreenSpaceShadows.cpp 开启了 OutEnvironment.SetDefine(TEXT("BEND_SSS"), 1)，则在 ScreenSpaceShadow.usf 会走 ScreenSpaceShadowsBendCS 的算法，用的还是原博客的 bend_sss_gpu.h。

如果没开，那就走 ScreenSpaceShadowsCS，和原始的 ContactShadow 一样，只不过是换成 Compute Shader 算的。

PC和移动端好像都有实现？使用宏做了区分

Bend 算法主要两点，第一使用 compute shader 并行处理深度计算，第二使用双边滤波估计边缘？

### 4.2

#### DispatchData 结构

```cpp
struct DispatchData {
    int WaveCount[3];           // [X, Y, Z] 三维调度维度
    int WaveOffset_Shader[2];   // [X, Y] 着色器中的偏移量
};
```

WaveCount[3] - 计算着色器调度的三维工作组数量
这个数组告诉GPU如何分派计算着色器：

	WaveCount[0]：固定为 inWaveSize（通常是64）
	
	这是GPU波前的大小，表示一个波前有多少个线程。
	
	WaveCount[1]：水平方向的波前数量
	
	例如：值为 10 表示横向有10个波前（640个线程）。
	
	WaveCount[2]：垂直方向的波前数量
	
	例如：值为 8 表示纵向有8个波前（512个线程）。

WaveOffset_Shader[2] - 相对于光源的像素偏移
这个值传递给着色器，告诉每个调度“从哪里开始计算”：

	WaveOffset_Shader[0]：X轴偏移（相对于光源位置）
	
	正数：光源右侧
	
	负数：光源左侧
	
	WaveOffset_Shader[1]：Y轴偏移（相对于光源位置）
	
	正数：光源下方
	
	负数：光源上方

	WaveOffset_Shader = [-320, 128]
	// 表示这个调度从光源左侧320像素、下方128像素处开始计算

#### DispatchList 结构

```cpp
struct DispatchList {
float LightCoordinate_Shader[4];  // 光源的完整坐标信息
DispatchData Dispatch[8];         // 最多8个调度
int DispatchCount;                // 实际需要的调度数量
};
```

LightCoordinate_Shader[4] - 光源的屏幕空间坐标
这是所有调度共享的全局数据，存储光源的关键信息。

```cpp
// [0] - 光源的屏幕X坐标（像素单位）
result.LightCoordinate_Shader[0] = ((inLightProjection[0] / xy_light_w) * +0.5f + 0.5f) * (float)inViewportSize[0];
// NDC坐标[-1,1] → 纹理坐标[0,1] → 像素坐标[0,width]

// [1] - 光源的屏幕Y坐标（像素单位，注意Y轴翻转）
result.LightCoordinate_Shader[1] = ((inLightProjection[1] / xy_light_w) * -0.5f + 0.5f) * (float)inViewportSize[1];
// 注意：-0.5f 是因为屏幕Y轴方向与NDC相反

// [2] - 光源的深度值
result.LightCoordinate_Shader[2] = inLightProjection[3] == 0 ? 0 : (inLightProjection[2] / inLightProjection[3]);
// 对于方向光（w=0）：深度为0
// 对于点光源（w≠0）：z/w 得到标准化深度

// [3] - 光源的方向标志
result.LightCoordinate_Shader[3] = inLightProjection[3] > 0 ? 1 : -1;
// +1：光源在相机前方
// -1：光源在相机后方（或是方向光）

例子：
LightCoordinate_Shader = [1200.5, 540.2, 0.85, 1.0]
// X = 1200.5 像素（屏幕右侧）
// Y = 540.2  像素（垂直居中）
// Z = 0.85   （深度值，0=远，1=近）
// W = 1.0    （在相机前方）
```

Dispatch[8] - 调度数组
最多存储8个 DispatchData，每个对应一次GPU调度。

DispatchCount - 实际调度数量
通常为 1-6个：

	光源在屏幕外：1-2个调度（只需要覆盖可见的扇形区域）
	
	光源在屏幕内：4-6个调度（需要覆盖周围4个象限）

### 4.6

阅读 GPU 侧实现，ComputeWavefrontExtents 用于计算每个运行组和每个线程应该开始采样的位置，后续深度采样在 WriteScreenSpaceShadow 中实现。先并行采样深度到共享内存 DepthData 中，其中每个组负责一块区域，每个线程负责这个区域中的一个（或者若干个固定间隔）像素，并行采样能大幅降低读显存的压力

TODO：阴影计算的实现？

### 4.13

编辑源码开放 IgnoreEdgePixels 的控制台参数，使用 r.ContactShadows.Bend.IgnoreEdgePixels 配置，设置为 True 之后可以缓解摩尔纹现象

TODO：阴影计算的实现？摩尔纹的具体成因？为什么开启之后就能缓解？

### 4.15

shadowing_depth 最后会写入 LDS DepthData 

sampling_depth 感觉不需要单独开一个数组，是不是为了内存对齐？

### 4.26

本质跟原本的 Contact Shadow 思路是一致的，需要对屏幕空间逐像素向光源方向（Bend 只处理方向光）做 ray marching，比深度，算交点，其中光照方向由 CPU 分 dispatch 的时候决定，其 artifact 还是由于物体厚度信息拿不到导致

对屏幕上每一个像素都计算一次沿着光线方向的遮挡情况，sampling_depth[0] 存像素起始深度，后续沿着光照方向的深度在共享内存 DepthData 中取出

具体到比较过程，我们先要明确 DepthData 中的数据含义，再看比较算法，最后确定导致逆光时 artifact 的原因

- DepthData的数据含义：

	在 bend SSS 的实现中，DepthData 并不是直接把深度缓冲搬进共享内存，而是经历了“采样 → 边缘检测 → 归一化”三步：每个线程先从 DepthTexture 读取相邻深度 depths.x 和 depths.y 来判断边缘，如果是边缘，根据 IgnoreEdgePixels 来判断是直接忽略边缘还是用点采样记录边缘深度；如果不是边缘，直接双线性插值做一个过滤；为了避免深度缓冲中的非线性深度值，随后把它变换为光空间下的深度再写入 DepthData，公式是 stored_depth = (shadowing_depth[i] - LightCoordinate.z) / sample_distance[i] 。这一步具体的数学原理是先减去 LightCoordinate.z 把深度转成相对光源参考量，再除以 sample_distance 消除不同采样距离带来的透视尺度差异，使来自不同线程、不同距离的样本都落到同一可比较坐标系中，后续 start_depth 与 DepthData[...] 的 depth_delta 比较才稳定且有一致物理意义


- 比较算法：

	首先是开头算了一个 depth_scale，这个变量实际上是把用户定义的 SurfaceThickness 参数归一化到比较空间，使得不同位置的像素都以 SurfaceThickness 为参考来估计物体厚度。接下来，在光线方向上用 abs(start_depth - DepthData[sample_index + i] * depth_scale)，在考虑 SurfaceThickness 的同时不断计算最小的 depth_delta，将其打包到 depth_delta_packed 中是因为 min 的时候可以顺便把 index 顺序也考虑进去，在相同 depth 的情况下，index 更小的优先。除了下标比较，depth_delta_packed 有提高采样率的作用，我们在光线方向上分别采样出4段的最小 depth_delta 统一压到 depth_delta_packed 中，最后算这四段平均的 depth_delta 作为最终结果，注意这不是屏幕空间的均匀滤波，而是沿着光线方向的深度平均，这一步是为了让阴影更加柔和。总的来说，该比较方法本质就是用沿光线方向各个像素中准确的前表面深度 + 根据 SurfaceThickness 估计出的后表面深度做求交判断


- 逆光 artifact 原因：

	用采样理论解释，我们是要用一个固定的 SurfaceThickness 去采样复杂的 3D 物体厚度，当场景物体厚度变化剧烈，即代表存在高频信息，一个固定的 SurfaceThickness 就会产生 artifact。其实顺光场景下也会出现这样的 artifact，只不过顺光时候阴影很少，所以并不明显


- 可行的解决方案：

	1. PPT中提到的 depth bias texture - 等价 depth buffer，没有用

    2. 降采样机制？不是，Bend 分支完全没走降采样，ScreenSpaceShadows 的 DownsampleFactor 不影响 CS 中的 Bend 分支

    3. 在逆光区（用 N dot L 判断）降低 SurfaceThickness - 试了，效果不好，保留了 artifact 的同时还几乎让 Contact Shadow 无效（因为 SurfaceThickness 降低了）

    4. 由于 3D 空间 model thickness 信息无法还原，我们在采样前无法模糊样本信号，所以只能考虑在采样后的逆光区做阴影 blend - 会导致阴影模糊

    5. 传统的 ScreenSpaceShadowRayCast 实现是沿着光源方向间隔几个像素采一次深度缓冲，采样频率更低，或许可以减轻 artifact？ - 尝试了关闭 UseHalfPixelOffset，降低采样率，稍微有点效果，但是没有根除，在处理高频信号时这样做反而会更加重 artifact

	6. 按照采样理论，可以引入随机偏移把 artifact 转换成噪声，或者按照原本 Contact Shadow 的算法计算边缘厚度 const float CompareTolerance = abs(RayDepth.z - RayStart.z) * Step * CompareToleranceScale

	7. 文档中提到的近距离用高 SurfaceThickness，远距离用低 SurfaceThickness

- 目前可以调节的参数：

	1. BacklitThicknessScale，根据背光程度缩放 Thickness，缩放公式为 inParameters.SurfaceThickness / thickness_scale

	2. BacklitNoLStart 和 BacklitNoLEnd，开始判断为背光的 NoL 值，把 BacklitNoLStart 调整为负数则能直接还原 artifact