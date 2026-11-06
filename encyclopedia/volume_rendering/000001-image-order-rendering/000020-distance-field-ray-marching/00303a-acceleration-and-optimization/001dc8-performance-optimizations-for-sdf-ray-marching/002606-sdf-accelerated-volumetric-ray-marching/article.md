# SDF-Accelerated Volumetric Ray Marching

Also called: Bipolar ray marcher, Distance fields (SDF), SDF-accelerated volumetric rendering, Signed Distance Field (SDF) Ray Marching for Volumetrics

## Overview
SDF-Accelerated Volumetric Ray Marching is a technique to accelerate volumetric ray marching by using a Signed Distance Field (SDF) to skip large empty spaces [S1, unofficial]. The SDF stores the distance to the nearest surface of the volume, allowing the ray to advance by that distance in empty space without missing intersections [S1, unofficial]. Unlike uniform sampling, this method spends samples only where clouds exist, avoiding expensive calculations for empty space [S1, unofficial]. It solves the problem of wasting GPU cycles calculating zero-value samples in sparse cloud volumes, which previously led to 'soft, undefined volumes' or 'horrible banding/noisy output' if sample counts were low [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin `rayStart`, ray direction `rayDir` (normalized), SDF texture `MapVolume`, fixed inner step size `_StepSize`, maximum steps `RAY_MAX_STEPS`.
*   **Outputs:** Accumulated color and transmittance `float4`.

**Data Structures:**
*   **SDF Texture:** A signed, half-precision floating point 3D texture [S1, unofficial].
    *   **Resolution:** 128x32x128 [S1, unofficial].
    *   **Sampling:** Non-uniform, stretched by a factor of 6 horizontally to cover a larger area [S1, unofficial].
    *   **Tiling:** Horizontal tiling for infinite extension [S1, unofficial].
    *   **Value:** Distance to the shell of the cloud volume [S1, unofficial].
    *   **Sign:** Negative if inside the volume, positive if outside [S1, unofficial].

**Algorithm:**
1.  **Initialization:** Initialize scattering/transmission accumulator `intScattTrans` to `float4(0, 0, 0, 1)`. Initialize distance along ray `t` to 0 [S1, unofficial].
2.  **Loop:** Iterate `u` from 0 to `RAY_MAX_STEPS` [S1, unofficial].
3.  **Position:** Calculate current ray position: `rayPos = rayStart + rayDir * t` [S1, unofficial].
4.  **SDF Evaluation:** Sample the SDF texture at `rayPos` to get `sdf`. Function: `float sdf = MapVolume(rayPos)` [S1, unofficial].
5.  **Inside Volume Check:** If `sdf < 0` (inside volume):
    *   Calculate extinction using `DensityFunction(sdf) = max(-sdf, 0)` [S1, unofficial].
    *   Calculate transmittance: `transmittance = exp(-extinction * _StepSize)` [S1, unofficial].
    *   Get luminance `luminance = Luminance(rayPos)` [S1, unofficial].
    *   Integrate scattering: `integScatt = luminance - luminance * transmittance` [S1, unofficial].
    *   Update accumulator: `intScattTrans.rgb += integScatt * intScattTrans.a` [S1, unofficial].
    *   Update alpha: `intScattTrans.a *= transmittance` [S1, unofficial].
    *   **Opaque Check:** If `intScattTrans.a < 0.003`, set `intScattTrans.a = 0.0` and break loop [S1, unofficial].
6.  **March Forward:** Update `t`. If inside volume (`sdf < 0`), step by `_StepSize`. If outside (`sdf >= 0`), step by `max(sdf, _StepSize)` [S1, unofficial].
7.  **Return:** Return `float4(intScattTrans.rgb, 1-intScattTrans.a)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def Raymarch(rayStart, rayDir, MapVolume, StepSize, RAY_MAX_STEPS):
    # Scattering in RGB, transmission in A
    intScattTrans = [0.0, 0.0, 0.0, 1.0]
    # Current distance along ray
    t = 0.0
    
    for u in range(RAY_MAX_STEPS):
        # Current ray position
        rayPos = rayStart + rayDir * t
        
        # Evaluate our signed distance field at the current ray position
        sdf = MapVolume(rayPos)
        
        # Only evaluate the cloud color if we're inside the volume
        if sdf < 0:
            # Density function: max(-sdf, 0)
            extinction = max(-sdf, 0.0)
            # Transmittance calculation
            transmittance = exp(-extinction * StepSize)
            
            # Get the luminance for the current ray position
            # [G] Luminance is assumed to be a function returning float3
            luminance = Luminance(rayPos)
            
            # Integrate scattering
            integScatt = luminance - luminance * transmittance
            intScattTrans[0] += integScatt[0] * intScattTrans[3]
            intScattTrans[1] += integScatt[1] * intScattTrans[3]
            intScattTrans[2] += integScatt[2] * intScattTrans[3]
            intScattTrans[3] *= transmittance
            
            # Opaque check
            if intScattTrans[3] < 0.003:
                intScattTrans[3] = 0.0
                break
        
        # March forward; step size depends on if we're inside the volume or not
        if sdf < 0:
            t += StepSize
        else:
            t += max(sdf, StepSize)
            
    return [intScattTrans[0], intScattTrans[1], intScattTrans[2], 1.0 - intScattTrans[3]]
```

**Parameters:**
*   `_StepSize`: Fixed step size used for ray marching when inside the cloud volume. Type: float (uniform) [S1, unofficial].
*   `RAY_MAX_STEPS`: Maximum number of ray march steps allowed per ray [S1, unofficial].

**Pitfalls:**
*   **Performance of SDF Generation:** Recalculating the SDF texture is expensive (approx 200ms on GTX1070 for a 3-pass compute shader) [S1, unofficial]. Solution: Use static cloud volumes. Movement is achieved by scrolling the volume texture rather than regenerating the SDF [S1, unofficial].

## Subtopics

### [SDF-Accelerated Volumetric Cloud Rendering](3700-sdf-accelerated-volumetric-cloud-rendering.md)
A technique to accelerate real-time volumetric cloud rendering by using a Signed Distance Field (SDF) to skip empty space and determine step sizes, allowing for high sample counts without performance loss. It solves the problem of wasting GPU cycles calculating zero-value samples in sparse cloud volumes, which previously led to 'soft, undefined volumes' or 'horrible banding/noisy output' if sample counts were low. Unlike uniform sampling, this method spends samples only where clouds exist, avoiding expensive calculations for empty space.

## Operating characteristics
*   **Time Complexity:** The method reduces the number of expensive volume evaluations by skipping empty space. Adaptively sampled distance fields (ADFs) achieve rendering speedups of up to 20 times compared to uniform sampling in complex scenes [S2].
*   **Memory Complexity:** Requires storage for the SDF texture. The example uses a 128x32x128 half-precision 3D texture [S1, unofficial].
*   **Measured Performance:** SDF generation takes about 200ms using a 3-pass compute shader on a GTX1070 [S1, unofficial].
*   **Strengths:** Avoids banding/noise by allowing high sample counts where it matters; skips empty space quickly [S1, unofficial].
*   **Limitations:** Requires static cloud volumes if SDF regeneration is too expensive; low-resolution SDFs may lose detail [S1, unofficial].
*   **When to use:** Real-time volumetric cloud rendering in VR to avoid banding/noise and improve performance by skipping empty space [S1, unofficial].
*   **When not to use:** Dynamic volumes where SDF regeneration cannot be avoided or amortized [S1, unofficial].

## Background
*   **Origin:** The technique is used in the UHawk VR engine for real-time volumetric cloud rendering in VR [S1, unofficial].
*   **History:** The method was described in a blog post by the UHawk VR team, referencing GPU Gems 2: Chapter 8 for the visualization of empty space skipping [S1, unofficial].
*   **Usage:** Used in the UHawk VR engine [S1, unofficial].

## Sources
[S1] Rendering volumetric clouds using signed distance fields, https://blog.uhawkvr.com/rendering/rendering-volumetric-clouds-using-signed-distance-fields/, fetched 2026-10-26 (unofficial)
[S2] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
