# Soft Shadow Ray Marching

Also called: Global Shading with Soft Shadows, MIP shadow, Penumbra Approximation, Quilez soft shadows, Soft Shadow Casting via Raymarching

## Overview
Soft Shadow Ray Marching computes a continuous shadow factor (0.0 to 1.0) for a surface point by ray marching towards a light source and accumulating the ratio of the distance to intervening geometry relative to the distance traveled [S1, unofficial][S2][S13, unofficial]. Unlike hard shadow ray marching, which returns a binary occlusion result, this method simulates the penumbra of an area light by tracking the minimum "cone ratio" or occlusion factor along the ray [S2][S17, unofficial]. It sets itself apart from [Shadow Casting Algorithm](1657-shadow-casting-algorithm.md) by providing smooth, alias-free penumbrae without requiring separate shadow map passes or low-pass filtering [S12, unofficial][S19, unofficial].

## How it works
**Inputs:**
*   `p`: The 3D surface point being shaded.
*   `lightPos`: The 3D position of the light source.
*   `map(p)`: The Signed Distance Function (SDF) of the scene, returning the distance to the nearest surface.
*   `k`: The softness/sharpness coefficient (dimensionless).
*   `maxDist`: The maximum distance to march (typically the distance to the light or a fixed limit).
*   `minDist`: The minimum starting distance (to prevent self-intersection).

**Outputs:**
*   `shadowFactor`: A float in $[0.0, 1.0]$, where 1.0 is fully lit and 0.0 is fully shadowed.

**Data Structures:**
No complex data structures are required.

**Algorithm:**
1.  **Initialization:**
    *   Calculate the light direction vector: $\vec{d} = \text{normalize}(\text{lightPos} - p)$ [S11, unofficial].
    *   Initialize the shadow factor: $res = 1.0$ [S3, unofficial].
    *   Initialize the traversal distance: $t = \text{minDist}$ [S17, unofficial]. A typical value for `minDist` is $0.01$ to avoid self-intersection artifacts [S4, unofficial].
    *   Determine the maximum march distance: $maxDist = \min(\text{length}(\text{lightPos} - p), \text{fixed\_limit})$ [S4, S8, unofficial]. Some implementations use a fixed limit (e.g., 30.0 or 60.0) regardless of light position to bound performance [S4, S8, unofficial].

2.  **Ray Marching Loop:**
    *   Iterate for a fixed number of steps (e.g., 32 to 64) or until $t \ge maxDist$ [S6, S7, S18, unofficial].
    *   Evaluate the SDF at the current sample point: $h = \text{map}(p + t \cdot \vec{d})$ [S5, unofficial].
    *   **Hard Occlusion Check:** If $h < \epsilon$ (where $\epsilon$ is a small threshold like $0.001$), the ray has hit an object; return $0.0$ [S4, S6, S7, S13, unofficial].
    *   **Soft Shadow Update:** Update the shadow factor by taking the minimum of the current value and the ratio of the distance to the surface ($h$) to the distance traveled ($t$), scaled by $k$:
        $$res = \min(res, k \cdot \frac{h}{t})$$
        This ratio tracks the minimum clearance relative to distance, creating a penumbra region [S3, unofficial][S12, unofficial][S17, unofficial].
    *   **Step Advancement:** Advance the ray by the distance to the nearest surface. To prevent overshooting small features or moving too fast, the step size is often clamped:
        $$t = t + \text{clamp}(h, \text{step\_min}, \text{step\_max})$$
        Typical clamping range is $[0.02, 0.25]$ [S6, unofficial], though some implementations use $[0.03, 0.8]$ [S8, unofficial].

3.  **Termination:**
    *   If the loop completes without hitting an object, return the clamped result: $\text{clamp}(res, 0.0, 1.0)$ [S4, S6, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_shadow_ray_march(p, light_pos, map_func, k=16.0, max_steps=32):
    """
    Computes soft shadow factor for point p towards light_pos.
    
    Args:
        p: vec3, surface point
        light_pos: vec3, light position
        map_func: function(vec3) -> float, SDF of the scene
        k: float, softness coefficient (default 16.0)
        max_steps: int, maximum number of march steps
        
    Returns:
        float, shadow factor in [0.0, 1.0]
    """
    # 1. Initialization
    light_dir = normalize(light_pos - p)
    res = 1.0
    t = 0.01  # minDist to avoid self-intersection [S4, unofficial]
    
    # Calculate max distance: distance to light or a fixed limit
    dist_to_light = length(light_pos - p)
    max_dist = min(dist_to_light, 30.0)  # 30.0 is a common fixed limit [S8, unofficial]
    
    # 2. Ray Marching Loop
    for i in range(max_steps):
        if t >= max_dist:
            break
            
        # Sample SDF
        h = map_func(p + t * light_dir)
        
        # Hard occlusion check
        if h < 0.001:
            return 0.0
            
        # Soft shadow update: track minimum cone ratio
        # Formula: res = min(res, k * h / t) [S3, unofficial][S12, unofficial][S17, unofficial]
        res = min(res, k * h / t)
        
        # Advance ray, clamping step size to prevent overshooting [S6, S8, unofficial]
        step = clamp(h, 0.02, 0.25)
        t += step
        
    # 3. Return clamped result
    return clamp(res, 0.0, 1.0)
```

**Parameters and Typical Values:**
*   `k` (Softness/Sharpness): Controls the width of the penumbra. Higher values produce sharper shadows (narrower penumbra), lower values produce softer shadows (wider penumbra) [S6, unofficial]. Typical values range from $4.0$ to $32.0$ [S7, S8, S13, unofficial].
*   `minDist` (Start Offset): Prevents the ray from starting inside the surface. Typical value: $0.01$ [S4, unofficial].
*   `maxDist` (Max March): Limits the shadow ray length. Can be the distance to the light or a fixed constant (e.g., $30.0$ or $60.0$) [S4, S8, unofficial].
*   `Step Clamp`: Limits the maximum step size to ensure fine geometry is not skipped. Typical range: $[0.02, 0.25]$ [S6, unofficial], though some implementations use $[0.03, 0.8]$ [S8, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Shadowing:** If the ray starts exactly at the surface point ($t=0$), the SDF query returns $0$, causing immediate false occlusion. A small `minDist` offset is required [S13, unofficial].
*   **Linear Drop-off Artifacts:** Using a linear denominator for the distance ratio (e.g., $h/t$) can cause severe shadow artifacts when light sources are very close to objects, as the ratio changes rapidly. Some implementations use a logarithmic denominator ($\log(t \cdot w + 1)$) to mitigate this [S13, unofficial].
*   **Precision:** If $h$ is very small but not below the threshold, the ratio $h/t$ becomes small, correctly darkening the shadow. However, floating-point precision issues near the surface can cause flickering if the step size is not clamped [G].

## Subtopics

### [Soft Shadow Visibility with Dithered Penumbra](4405-soft-shadow-visibility-with-dithered-penumbra.md)
Calculates soft shadow visibility (0.0 to 1.0) towards a directional light (e.g., the sun) by marching along the light direction, using a dithered rotation of the light vector per pixel to create a penumbra that averages out with supersampling. It sets itself apart from standard soft shadow ray marching by normalizing the penumbra calculation by $t \cdot \text{sunY}$ rather than distance alone, preventing false heavy shadows on flat ground where clearance is proportional to distance. Choose this variant when rendering directional lights over large, flat surfaces where standard distance-based penumbrae would incorrectly darken the ground plane.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of ray march steps. Typically fixed between 32 and 64 steps per shadow ray [S6, S7, unofficial].
*   **Memory Complexity:** $O(1)$. No additional memory buffers are required; it uses the same SDF queries as the primary ray march [S19, unofficial].
*   **Performance:** Soft shadows are "almost free" compared to hard shadows in SDF ray marching because they require no additional ray casts or texture lookups, only a few extra arithmetic operations per step [S9, S10, unofficial].
*   **Strengths:**
    *   Produces smooth, alias-free penumbrae [S13, unofficial].
    *   Works with dynamic geometry without requiring shadow map recomputation [S12, unofficial].
    *   Integrates seamlessly with existing SDF ray marching pipelines [S19, unofficial].
*   **Limitations:**
    *   Approximates area lights; the softness is controlled by a single scalar $k$ rather than a true light radius, which may not match physical accuracy for all light sizes [S11, unofficial].
    *   Performance is linear with the number of lights; multiple lights require multiple shadow ray marches [S11, unofficial].
    *   Can suffer from artifacts if the light is very close to the surface due to the linear ratio assumption [S11, unofficial].

## Background
The technique was popularized by Inigo Quilez, who described it as a way to obtain soft shadows with a penumbra region using distance fields by inspecting the distance $d$ from the ray to surrounding geometry and the traversal distance $t$ [S13, unofficial]. The method leverages the global geometric information provided by SDFs to approximate the visibility of an area light without the overhead of shadow maps or multiple ray samples [S12, unofficial][S19, unofficial].

## Sources
[S1] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
[S3] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S4] a1k0n/asciitracing, https://github.com/a1k0n/asciitracing, fetched 2026-10-26 (unofficial)
[S5] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26 (unofficial)
[S6] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S7] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S8] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S9] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S10] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S11] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1819/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S12] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S13] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S14] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S15] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S16] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S17] conradplake/ray-marching-shader, https://github.com/conradplake/ray-marching-shader, fetched 2026-10-27 (unofficial)
[S18] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
[S19] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S20] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S21] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S22] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
