# Volumetric Light Shafts (URP Raymarching)

Also called: God Rays, Light Shafts, VolumetricLightURP

## Overview
Volumetric Light Shafts (URP Raymarching) is a post-processing or overlay technique for rendering volumetric light shafts (crepuscular rays) in the Unity Universal Render Pipeline (URP) by raymarching from the camera through a fixed-distance volume, modulated by the main light's realtime shadows [S2, unofficial]. It solves the problem of simulating atmospheric fog beams and god rays in real-time 3D scenes where full volumetric path tracing is too expensive [S2, unofficial]. Unlike its sibling [Screen-Space God Rays](3981-screen-space-god-rays.md), which uses radial blur on an occlusion map, this method performs actual 3D raymarching with shadow sampling, providing geometrically consistent light shafts that respect scene depth and shadows, at the cost of higher performance overhead [S2, unofficial]. It builds on [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) by using a fixed step size and exponential absorption logic, but specifically targets light shafts from a single directional light [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera world position, fragment world position (from vertex shader), main light position (`_LightPos`), main light color/intensity (`_LightColor`), density (`_Density`), step count (`_StepCount`), max distance (`_MaxDistance`), scattering multiplier (`_Scattering`), and base color (`_BaseColor`).
*   **Outputs:** Accumulated light color and alpha for the current pixel.

**Data Structures:**
*   **Shader Properties (CBUFFER):**
    *   `_BaseColor`: `float4` (RGBA), default `(1,1,1,1)`.
    *   `_Density`: `Float`, default `0.5`, range `[0, 1]`.
    *   `_StepCount`: `Int`, default `64`, range `[16, 128]`.
    *   `_MaxDistance`: `Float`, default `20.0`.
    *   `_Scattering`: `Float`, default `1.2`.
    *   `_LightPos`: `float4` (xyz: world position, w: 1.0).
    *   `_LightColor`: `float4` (RGB: color * intensity, A: unused).
*   **Vertex Shader Output:**
    *   `positionHCS`: Clip space position.
    *   `positionWS`: World space position.
*   **Fragment Shader State:**
    *   `rayOrigin`: Camera world position.
    *   `rayDir`: Normalized vector from camera to fragment world position.
    *   `accum`: Accumulated light contribution, initialized to `0.0`.

**Parameters:**
*   `g` (Henyey-Greenstein asymmetry parameter): `0.35` (hardcoded) [S2, unofficial].
*   `stepSize`: Calculated as `_MaxDistance / steps` [S2, unofficial].

**Numbered Steps:**
1.  **Vertex Shader:** Transform input object space position to clip space and world space. Pass world space position to fragment shader via `TEXCOORD0` [S2, unofficial].
2.  **Fragment Shader Initialization:**
    *   Get camera world position (`camPos`) [S2, unofficial].
    *   Get fragment world position (`posWS`) from `TEXCOORD0` [S2, unofficial].
    *   Calculate ray direction: `rayDir = normalize(posWS - camPos)` [S2, unofficial].
    *   Initialize `accum = 0.0` [S2, unofficial].
    *   Determine `steps = max(_StepCount, 1)` to prevent division by zero [S2, unofficial].
    *   Calculate `stepSize = _MaxDistance / steps` [S2, unofficial].
3.  **Raymarching Loop:** Iterate `i` from `0` to `steps - 1`:
    *   Calculate distance: `dist = i * stepSize` [S2, unofficial].
    *   Calculate sample position: `samplePos = rayOrigin + rayDir * dist` [S2, unofficial].
    *   **Shadow Sampling:**
        *   Transform `samplePos` to shadow coordinates: `shadowCoord = TransformWorldToShadowCoord(samplePos)` [S2, unofficial].
        *   Sample main light realtime shadow: `shadow = MainLightRealtimeShadow(shadowCoord)` [S2, unofficial].
    *   **Light Direction:** Calculate vector from sample to light: `lightDir = normalize(_LightPos.xyz - samplePos)` [S2, unofficial].
    *   **Phase Function:** Calculate scattering angle cosine: `cosTheta = dot(rayDir, lightDir)` [S2, unofficial].
        *   Compute Henyey-Greenstein phase function:
            $$P(\theta) = \frac{1 - g^2}{(1 + g^2 - 2g \cos\theta)^{1.5}}$$
            where $g = 0.35$ [S2, unofficial].
    *   **Accumulation:** Add to `accum`:
        $$accum += shadow \times P(\theta) \times _Density \times stepSize$$
        [S2, unofficial].
4.  **Final Color:**
    *   Calculate `lightAmount = accum` [S2, unofficial].
    *   Output color: `_LightColor.rgb * lightAmount * _Scattering` [S2, unofficial].
    *   Output alpha: `saturate(lightAmount)` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
struct Varyings {
    float4 positionHCS : SV_POSITION;
    float3 positionWS : TEXCOORD0;
};

Varyings vert(float3 positionOS : POSITION) {
    Varyings OUT;
    // Unity URP helper to get world space and clip space positions
    VertexPositionInputs pos = GetVertexPositionInputs(positionOS);
    OUT.positionHCS = pos.positionCS;
    OUT.positionWS = pos.positionWS;
    return OUT;
}

# Fragment Shader
float HenyeyGreenstein(float cosTheta, float g) {
    float g2 = g * g;
    return (1.0 - g2) / pow(1.0 + g2 - 2.0 * g * cosTheta, 1.5);
}

float4 frag(Varyings IN) : SV_TARGET {
    // Get camera world position
    float3 camPos = GetCameraWorldPosition(); // [G] Standard URP helper or equivalent
    
    float3 rayDir = normalize(IN.positionWS - camPos);
    float accum = 0.0;
    
    // Clamp step count to prevent division by zero
    int steps = max(_StepCount, 1);
    float stepSize = _MaxDistance / steps;
    
    for (int i = 0; i < steps; i++) {
        float dist = i * stepSize;
        float3 samplePos = camPos + rayDir * dist;
        
        // Shadow sampling
        float4 shadowCoord = TransformWorldToShadowCoord(samplePos);
        float shadow = MainLightRealtimeShadow(shadowCoord);
        
        // Light direction
        float3 lightDir = normalize(_LightPos.xyz - samplePos);
        
        // Phase function
        float cosTheta = dot(rayDir, lightDir);
        float phase = HenyeyGreenstein(cosTheta, 0.35);
        
        // Accumulation
        accum += shadow * phase * _Density * stepSize;
    }
    
    float lightAmount = accum;
    float3 finalColor = _LightColor.rgb * lightAmount * _Scattering;
    float alpha = saturate(lightAmount);
    
    return float4(finalColor, alpha);
}
```

**Edge Cases and Pitfalls:**
*   If `_StepCount` is less than 1, it is clamped to 1 to prevent division by zero or invalid loop behavior [S2, unofficial].
*   The method only supports a single directional light (the "main light") [S2, unofficial].
*   It does not support complex volume textures or multiple light sources [S2, unofficial].
*   Performance cost is significant: adds ~2.30 ms to a 5.0 ms frame in benchmarked scenarios [S1, unofficial].

## Subtopics

### [Screen-Space God Rays](3981-screen-space-god-rays.md)
A cheaper post-processing alternative using radial blur on an occlusion map. It fails with multiple lights or off-screen lights but is faster.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of steps (`_StepCount`). Each step involves a shadow coordinate transform and a shadow texture sample.
*   **Memory Complexity:** $O(1)$ per pixel for accumulation, plus standard shadow map memory.
*   **Performance:** Adds approximately 2.30 ms to a 5.0 ms frame in benchmarked tests [S1, unofficial].
*   **Strengths:** Geometrically consistent with scene depth and shadows; supports dynamic light movement and density changes via C# controller.
*   **Limitations:** Only works with the main directional light; no support for multiple lights or volumetric textures; performance scales linearly with step count and resolution.
*   **When to use:** Stylized atmospheric lighting, fog beams, or god rays where shadow accuracy is important and performance budget allows.
*   **When not to use:** Scenes with multiple light sources, off-screen lights, or when performance is critical and screen-space approximation is acceptable.

## Background
The technique is implemented as a custom shader and C# controller for the Unity Universal Render Pipeline (URP) [S2, unofficial]. It is designed for fog beams, atmospheric lighting, and stylized god ray effects [S2, unofficial]. The implementation uses the Henyey-Greenstein phase function with a hardcoded asymmetry parameter $g=0.35$ [S2, unofficial]. The method is distinct from voxel-based god ray implementations (like those in the voxel engine benchmark) which march shafts from surfaces brighter than the sky [S1, unofficial].

## Sources
[S1] DanielWLiu07/voxel-engine, https://github.com/DanielWLiu07/voxel-engine, fetched 2026-10-26 (unofficial)
[S2] Ninedark9/Volumetric-Light-Shafts-, https://github.com/Ninedark9/Volumetric-Light-Shafts-, fetched 2026-10-26 (unofficial)
