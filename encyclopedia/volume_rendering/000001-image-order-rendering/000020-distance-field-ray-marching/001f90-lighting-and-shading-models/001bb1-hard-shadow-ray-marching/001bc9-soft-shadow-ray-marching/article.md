# Soft Shadow Ray Marching

Also called: MIP shadow, Quilez soft shadows, SDF Shadows, Soft March, Soft Shadow Calculation via SDF, Soft Shadow Casting via Raymarching, Soft Shadows, Soft Shadows via SDF

## Overview
Soft Shadow Ray Marching computes a soft shadow factor along a ray by accumulating the ratio of the Signed Distance Field (SDF) height to the ray step distance, simulating area light penumbra [S1, S3, S8]. Unlike [Hard Shadow Ray Marching](1087-hard-shadow-ray-marching.md), which returns a binary occlusion result, this method attenuates illumination based on how close the ray passes to intervening geometry relative to the light's size [S5, S8]. It differs from [Shadow Casting Algorithm](1657-shadow-casting-algorithm.md) by leveraging the global geometric information inherent in the SDF, requiring no separate shadow map pass or rasterization overhead [S9].

## How it works
**Inputs:**
*   Surface point $p$ (position on the zero-isosurface).
*   Light direction $L$ (normalized vector from $p$ to light source).
*   Scene SDF function $f(x)$ returning distance to nearest geometry.
*   Light size parameter $L_{size}$ (or sharpness coefficient $k$).
*   Maximum shadow distance $D_{max}$ (optional, for performance).
*   Maximum iteration count $N_{max}$ (optional, for performance).

**Outputs:**
*   Shadow factor $s \in [0, 1]$, where 1 is fully lit and 0 is fully shadowed.

**Data Structures:**
*   No complex structures required. Standard floating-point variables for distance, position, and accumulation.

**Parameters:**
*   $k$ (or $L_{size}$): Controls shadow hardness/penumbra width. Typical values: $k=12.0$ [S1], $k=16.0$ [S5], $k=20.0$ [S2], or $L_{size}=0.03$ [S3].
*   $D_{max}$: Maximum ray distance. Typical value: $60.0$ [S1, S3].
*   $N_{max}$: Maximum steps. Typical value: $64$ [S4], $128$ [S3], or $60$ [S1].
*   $\epsilon$: Minimum step size or occlusion threshold. Typical value: $0.0001$ [S1, S3] or $0.001$ [S2].

**Algorithm:**
1.  Initialize accumulated shadow factor $res = 1.0$.
2.  Initialize current distance along ray $t = \epsilon$ (to avoid self-intersection) [S3, S8].
3.  Loop for $i$ from $0$ to $N_{max}-1$:
    a.  Evaluate sample point $p_{sample} = p + t \cdot L$.
    b.  Query SDF: $h = f(p_{sample})$.
    c.  **Hard Occlusion Check:** If $h < \epsilon$, the ray has intersected geometry. Return $0.0$ [S2, S3, S4, S8].
    d.  **Penumbra Calculation:** Update $res = \min(res, k \cdot \frac{h}{t})$ [S1, S4, S7, S8].
        *   *Note:* Some implementations use a logarithmic denominator to prevent artifacts when lights are near objects: $res = \min(res, \frac{h}{\log(t \cdot w + 1.0)})$ where $w$ is a sharpness parameter [S6].
    e.  **Step Advance:** $t = t + h$ [S3, S8].
    f.  **Termination Check:** If $t > D_{max}$ or $res < \epsilon$, break loop [S1, S3].
4.  Return $\text{clamp}(res, 0.0, 1.0)$ [S1, S3, S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_shadow_ray_march(p, light_dir, scene_sdf, k=12.0, max_dist=60.0, max_steps=64):
    """
    Computes soft shadow factor using SDF ray marching.
    
    Args:
        p: vec3, surface point
        light_dir: vec3, normalized direction to light
        scene_sdf: function, returns distance to nearest geometry
        k: float, shadow hardness coefficient (inverse of light size)
        max_dist: float, maximum ray length
        max_steps: int, maximum iterations
        
    Returns:
        float: shadow factor in [0, 1]
    """
    res = 1.0
    t = 0.001  # epsilon to avoid self-intersection [S3, S8]
    
    for _ in range(max_steps):
        if t > max_dist:
            break
            
        # Sample point along the ray
        sample_point = p + t * light_dir
        
        # Query distance field
        h = scene_sdf(sample_point)
        
        # Hard occlusion: if we hit geometry closely, it's fully shadowed
        if h < 0.0001:
            return 0.0
            
        # Accumulate soft shadow factor
        # Quilez formula: min(res, k * h / t) [S1, S4, S7, S8]
        # Alternative logarithmic form to prevent near-light artifacts [S6]:
        # res = min(res, h / log(t * w + 1.0))
        res = min(res, k * h / t)
        
        # Advance ray by the safe distance
        t += h
        
        # Early exit if fully shadowed
        if res < 0.001:
            break
            
    # Clamp result to valid range
    return max(0.0, min(1.0, res))
```

**Edge Cases and Pitfalls:**
*   **Self-Shadowing:** The initial step $t$ must be greater than zero (typically $\epsilon \approx 0.001$) to prevent the surface point from occluding itself [S3, S8].
*   **Near-Light Artifacts:** A linear drop-off ($h/t$) can cause severe artifacts when light sources are placed very close to objects [S6]. Using a logarithmic denominator or capping the ratio mitigates this [S6].
*   **Precision:** If $h$ is extremely small but not below the threshold, the ratio $h/t$ may underflow or cause instability. The threshold check $h < \epsilon$ handles this by returning hard shadow [S1, S3].
*   **Alias Artifacts:** Hard shadows exhibit aliasing; soft shadows naturally smooth these transitions due to the penumbra calculation [S8].

## Subtopics

### Logarithmic Soft Shadows
Uses $\log(t \cdot w + 1.0)$ in the denominator instead of linear $t$ to prevent artifacts when lights are near geometry [S6]. Choose when lights are dynamic or close to surfaces.

### Scale-Based Penumbra
Uses a `scale` parameter directly in `min(shadow, scale * d/t)` to control penumbra width visually [S8]. Choose for artistic control over shadow softness independent of physical light size.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of ray march steps. Typically $N \approx 60-128$ [S1, S3].
*   **Memory Complexity:** $O(1)$ auxiliary memory. No shadow maps or buffers required [S9].
*   **Performance:** Adds minimal overhead to hard shadow calculation, as it reuses the same ray march loop [S5, S9]. Performance scales with scene density; denser geometry may require more steps or earlier termination.
*   **Strengths:** No separate pass required; works with any SDF; naturally anti-aliased; supports dynamic geometry without update costs [S8, S9].
*   **Limitations:** Approximate; does not simulate complex light transport (e.g., multiple bounces); quality depends on step size and $k$ parameter tuning [S6, S8].

## Background
The technique was popularized by Inigo Quilez, who demonstrated that SDFs provide global scene information useful for soft shadows without additional data structures [S8]. It is a standard feature in raymarched SDF renderers, often cited alongside ambient occlusion as a "free" shading effect [S9]. Variations include logarithmic adjustments for near-light stability [S6] and scale-based penumbra control [S8].

## Sources
[S1] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] a1k0n/asciitracing, https://github.com/a1k0n/asciitracing, fetched 2026-10-26 (unofficial)
[S3] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S4] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S5] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S6] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27
[S7] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S8] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27
[S9] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
