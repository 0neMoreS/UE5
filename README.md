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