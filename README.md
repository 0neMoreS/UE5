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
    <img src="./blog_img/Cascade+Bend.png" style="width: 100%;" />
    <p>Cascade + Bend</p>
  </div>
</div>

### 3.29

查看 RenderScreenSpaceShadows 的调用点

1. FDeferredShadingSceneRenderer::RenderDeferredShadowProjections，这里是我手动添加的调用点。调用链路：FDeferredShadingSceneRenderer::Render -> FDeferredShadingSceneRenderer::RenderLights -> FDeferredShadingSceneRenderer::RenderDeferredShadowProjections -> RenderScreenSpaceShadows

2. FMobileSceneRenderer::RenderMobileShadowProjections，这里是 UE 移动端的调用点。调用链路：FMobileSceneRenderer::Render -> FMobileSceneRenderer::RenderMobileShadowProjections -> RenderScreenSpaceShadows

注意在这边开启之后，LightRendering.cpp 那边就要关掉 Contact Shadow 的渲染，不然会重复渲染两遍 Contact Shadow

### 3.30

RenderScreenSpaceShadowsBend 的具体实现

首先调用原博客中提供的 bend_sss_cpu.h 进行数据准备，再调用到 ScreenSpaceShadow.usf 进行实际的计算。

在 ScreenSpaceShadow.usf 中，如果在 ScreenSpaceShadows.cpp 开启了 OutEnvironment.SetDefine(TEXT("BEND_SSS"), 1)，则在 ScreenSpaceShadow.usf 会走 ScreenSpaceShadowsBendCS 的算法，用的还是原博客的 shader。

如果没开，那就走 ScreenSpaceShadowsCS，和原始的 ContactShadow 一样，只不过是换成 Compute Shader 算的。

是否是移动端好像都有实现？使用宏做了区分