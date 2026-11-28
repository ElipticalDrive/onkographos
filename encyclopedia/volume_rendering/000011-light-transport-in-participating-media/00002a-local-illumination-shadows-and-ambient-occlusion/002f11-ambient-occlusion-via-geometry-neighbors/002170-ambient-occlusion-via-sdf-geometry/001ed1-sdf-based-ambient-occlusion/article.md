# SDF-based Ambient Occlusion

Also called: Signed Distance Field AO, Distance Field Ambient Occlusion, Raymarched AO

## Overview
SDF-based Ambient Occlusion estimates local occlusion by sampling a Signed Distance Field (SDF) along the surface normal at increasing distances, weighting closer samples more heavily to simulate ambient light blockage [S2, S4, S3, unofficial]. It approximates ambient occlusion by using the distance field itself to simulate light occlusion based on object proximity, avoiding the unrealistic results of Screen Space Ambient Occlusion (SSAO) which relies on depth buffers [S5]. Unlike SSAO, this method leverages the global geometric information stored in the SDF, providing more accurate occlusion in scenes with complex geometry or self-occlusion that might be missed by screen-space techniques [S5].

## How it works
The method operates on a 3D grid where each position stores the distance to the nearest object surface [S1, unofficial]. Negative values indicate the grid location is inside the object, and positive values indicate it is outside [S1, unofficial]. The core algorithm samples the SDF along the surface normal $\mathbf{n}$ at a surface point $\mathbf{p}$, accumulating occlusion based on the difference between the sample distance and the actual SDF value [S4, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   $\mathbf{p}$: Surface point in world space.
    *   $\mathbf{n}$: Surface normal at $\mathbf{p}$.
    *   $map(\mathbf{x})$: SDF evaluation function returning the signed distance to the nearest geometry at $\mathbf{x}$.
    *   $N_{samples}$: Number of samples to take along the normal (typically 4–8) [S2, S3, S4, unofficial].
    *   $d_{max}$: Maximum sampling distance (typically 0.25) [S3, unofficial].
    *   $bias$: Small offset to prevent self-intersection (acne) [G].
*   **Outputs:**
    *   $AO$: Scalar occlusion factor in $[0, 1]$, where 1 is fully lit and 0 is fully occluded.

### Data Structures
*   **SDF Grid:** A 3D array where each position stores the distance to the nearest object surface [S1, unofficial].

### Parameters
Typical parameter values found in implementations:
*   $N_{samples}$: 4 to 8 [S2, S3, S4, unofficial].
*   $d_{max}$: 0.25 units [S3, unofficial].
*   Weight decay factor: 0.85 or 0.92 [S2, S4, unofficial].
*   Final scale factor: 0.6 or 3.0 (multiplier) [S2, S4, unofficial].

### Algorithm Steps
1.  **Initialization:** Initialize accumulated occlusion $O = 0.0$ and weight $w = 1.0$ [S2, unofficial].
2.  **Sampling Loop:** For $i$ from 0 to $N_{samples} - 1$:
    a. Calculate sample distance $len_i$. Implementations vary in spacing:
        *   Quadratic spacing: $len_i = 0.01 + 0.02 \cdot i^2$ [S2, unofficial].
        *   Linear spacing: $len_i = \frac{d_{max}}{N_{samples}} \cdot i$ [S3, unofficial].
        *   Linear spacing (5 samples): $h = 0.01 + 0.12 \cdot \frac{i}{4.0}$ [S4, unofficial].
    b. Calculate sample point $\mathbf{p}_i = \mathbf{p} + \mathbf{n} \cdot (len_i + bias)$ [G].
    c. Query SDF: $d_i = map(\mathbf{p}_i)$ [S2, S3, S4, unofficial].
    d. Calculate occlusion contribution. If $d_i < len_i$, the geometry is closer than the sample point, contributing to occlusion [S2, unofficial].
        *   Method A (Weighted Difference): $O += (len_i - d_i) \cdot w$ [S2, S4, unofficial].
        *   Method B (Clamped Difference): $O += \max(-(d_i - len_i), 0.0)$ [S3, unofficial].
    e. Decay weight: $w *= \text{decay\_factor}$ (e.g., 0.85 or 0.92) [S2, S4, unofficial].
3.  **Normalization and Clamping:**
    *   Method A: $AO = 1.0 - \text{clamp}(scale \cdot O, 0.0, 1.0)$ where $scale = 0.6$ [S2, unofficial].
    *   Method B: $AO = \frac{O}{d_{max} \cdot N_{samples} \cdot 4.0}$ [S3, unofficial].
    *   Method C: $AO = \text{clamp}(1.0 - 3.0 \cdot O \cdot u_{ao}, 0.0, 1.0)$ where $u_{ao}$ is a user-defined strength multiplier [S4, unofficial].

### Pseudocode
The following pseudocode implements the weighted difference approach with quadratic spacing [S2, unofficial], as it is the most detailed reference in the sources.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_ambient_occlusion(p, n, map_func, num_samples=8, max_dist=0.25, bias=0.001, scale=0.6, decay=0.85):
    """
    Calculate SDF-based Ambient Occlusion.
    
    Args:
        p: Surface point (vec3)
        n: Surface normal (vec3)
        map_func: Function returning signed distance to geometry
        num_samples: Number of samples along the normal
        max_dist: Maximum sampling distance
        bias: Offset to prevent self-intersection
        scale: Final scaling factor for occlusion
        decay: Weight decay factor
        
    Returns:
        AO factor in [0, 1]
    """
    occ = 0.0
    weight = 1.0
    
    for i in range(num_samples):
        # Quadratic spacing for sample distance [S2, unofficial]
        len_i = 0.01 + 0.02 * (i * i)
        
        # Sample point along normal with bias [G]
        sample_point = p + n * (len_i + bias)
        
        # Query SDF [S2, unofficial]
        dist = map_func(sample_point)
        
        # Accumulate occlusion if geometry is closer than sample point [S2, unofficial]
        if dist < len_i:
            occ += (len_i - dist) * weight
        
        # Decay weight for subsequent samples [S2, unofficial]
        weight *= decay
        
    # Final AO calculation [S2, unofficial]
    ao = 1.0 - clamp(scale * occ, 0.0, 1.0)
    
    return ao
```

### Edge Cases and Pitfalls
*   **Self-Intersection (Acne):** Floating-point precision errors can cause the sample point to be slightly inside the geometry, resulting in negative distances and incorrect occlusion. A small bias offset along the normal is required [G].
*   **Precision:** The accuracy of the AO depends on the resolution of the SDF. Low-resolution SDFs may produce blocky or inaccurate occlusion [G].
*   **Boundary Conditions:** At the boundaries of the SDF grid, queries may return undefined or large values. Proper handling of grid boundaries is necessary [G].
*   **Parameter Tuning:** The scale, decay, and number of samples significantly affect the visual result [G].

## Subtopics

### Medial Field Ambient Occlusion (MFAO)
MFAO is an efficient alternative to SSAO for approximating ambient occlusion using an O(1) medial field [S6, unofficial]. It measures local shape thickness of the shape complement to derive a proxy for local occlusion at a surface point, without viewpoint dependence, sampling, or filtering [S6, unofficial]. It sets itself apart from standard SDF-based AO by avoiding ray marching/sampling entirely, instead evaluating a learned medial field directly [S6, unofficial]. It should be chosen when an O(1) medial field is already available and computational cost must be minimized [S6, unofficial]. Like SSAO, it is local and does not consider distant geometry [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples per pixel [G].
*   **Memory Complexity:** Depends on the SDF storage. A dense 3D grid requires $O(V)$ memory where $V$ is the volume of the grid [G].
*   **Strengths:** Provides accurate occlusion in global context, handles self-occlusion well, and works well with dynamic geometry if the SDF is updated [S5].
*   **Limitations:** Requires an SDF representation of the scene, which can be memory-intensive for large scenes [S5].

## Background
The technique was introduced by Evans, who embedded objects into a 3D grid of signed distance fields and calculated occlusion via a heuristic combining values sampled at increasing distances along the surface normal [S1, unofficial]. Wright expanded on this approach by substituting the ad hoc heuristic with cone tracing, approximated by sphere tracing [S1, unofficial]. The method is widely used in real-time rendering, particularly in voxel-based global illumination systems, to provide ambient occlusion that is consistent with the global geometry [S1, unofficial][S5].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S3] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S4] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S5] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-07
[S6] Deep Medial Fields, https://arxiv.org/abs/2106.03804v1, fetched 2026-10-26 (unofficial)
