# SDF-based Ambient Occlusion

Also called: Signed Distance Field AO, Distance Field Ambient Occlusion, Raymarched AO

## Overview
SDF-based Ambient Occlusion (AO) estimates local occlusion by sampling a Signed Distance Field (SDF) along the surface normal at increasing distances, weighting closer samples more heavily to simulate ambient light blockage [S1, S2, S3, S4, unofficial]. Unlike Screen-Space Ambient Occlusion (SSAO), which relies on depth buffers and can produce unrealistic artifacts, this method uses the geometric proximity information inherent in the distance field to approximate how occluded a surface is from surrounding geometry [S5]. It builds on the parent method [Ambient Occlusion via SDF/Geometry](2527-ambient-occlusion-via-sdf-geometry.md) by utilizing the SDF's explicit distance queries rather than neighbor lookups or screen-space heuristics.

## How it works
### Inputs and Outputs
- **Input:**
  - Surface point $\mathbf{p}$ (world or local space).
  - Surface normal $\mathbf{n}$ at $\mathbf{p}$.
  - SDF query function $SDF(\mathbf{x})$ returning the signed distance to the nearest surface.
  - Parameters: Number of samples $N$, maximum sampling distance $D_{max}$, weight decay factor $\alpha$, and strength multiplier $k$.
- **Output:**
  - Ambient Occlusion factor $AO \in [0, 1]$, where 0 is fully occluded and 1 is fully lit.

### Data Structures
The core data structure is a 3D grid (or implicit function) storing the signed distance to the nearest object surface [S1, unofficial].
- **Layout:** Each grid position (or implicit coordinate) stores a single floating-point value $d$.
- **Interpretation:** Negative values indicate the point is inside the object; positive values indicate the point is outside [S1, unofficial].
- **Access:** The SDF is queried via trilinear interpolation if stored in a grid, or via analytical distance functions if stored implicitly [S4, unofficial].

### Algorithm Steps
The method calculates the occlusion factor by accumulating the difference between the sample distance along the normal and the actual distance to geometry found by the SDF [S3, S4, unofficial].

1.  **Initialize:** Set accumulated occlusion $O = 0.0$ and weight $w = 1.0$.
2.  **Loop:** For $i$ from $0$ to $N-1$:
    a.  **Calculate Sample Distance:** Determine the distance $len_i$ along the normal for the current sample. Sources vary on the spacing:
        - Quadratic spacing: $len_i = D_{start} + D_{step} \cdot i^2$ [S2, unofficial].
        - Linear spacing: $len_i = D_{start} + D_{step} \cdot i$ [S3, S4, unofficial].
    b.  **Query SDF:** Calculate the sample point $\mathbf{p}_i = \mathbf{p} + len_i \cdot \mathbf{n}$. Query the SDF to get the distance to the nearest geometry: $d_i = SDF(\mathbf{p}_i)$ [S2, S3, unofficial].
    c.  **Accumulate Occlusion:** If the geometry is closer than the sample point ($d_i < len_i$), it contributes to occlusion. The contribution is the difference $(len_i - d_i)$ weighted by $w$ [S2, S3, S4, unofficial].
        $$ O \leftarrow O + \max(0.0, len_i - d_i) \cdot w $$
        *Note: Some implementations use $\max(-(d_i - len_i), 0.0)$ which is equivalent [S3, unofficial].*
    d.  **Decay Weight:** Update the weight for the next sample: $w \leftarrow w \cdot \alpha$ [S2, S4, unofficial].
3.  **Finalize:** Convert the accumulated occlusion $O$ into a final AO factor.
    - Common formulation: $AO = 1.0 - \text{clamp}(k \cdot O, 0.0, 1.0)$ [S2, unofficial].
    - Alternative normalization: $AO = \text{clamp}(1.0 - k \cdot O, 0.0, 1.0)$ [S4, unofficial].

### Formulas
The general accumulation formula, combining the specific parameters from [S2, unofficial], is:

$$ O = \sum_{i=0}^{N-1} \max(0.0, len_i - SDF(\mathbf{p} + len_i \cdot \mathbf{n})) \cdot \alpha^i $$

Where:
- $len_i = D_{start} + D_{step} \cdot i^2$ (Quadratic spacing) [S2, unofficial]
- $len_i = D_{start} + D_{step} \cdot i$ (Linear spacing) [S3, S4, unofficial]
- $AO = 1.0 - \text{clamp}(k \cdot O, 0.0, 1.0)$ [S2, unofficial]

### Parameters and Typical Values
- **$N$ (AO_SAMPLES):** Number of samples. Typical values: 4 [S3, unofficial], 5 [S4, unofficial], 8 [S2, unofficial].
- **$D_{start}$:** Initial offset to avoid self-intersection or noise at the surface. Typical value: $0.01$ [S2, S3, S4, unofficial].
- **$D_{step}$:** Step size increment.
    - Quadratic: $0.02$ [S2, unofficial].
    - Linear: $0.12 / (N-1)$ or similar to reach max distance [S4, unofficial].
- **$\alpha$ (AO_WEIGHT_DECAY):** Weight decay factor. Typical values: $0.85$ [S2, unofficial], $0.92$ [S4, unofficial].
- **$k$ (AO_SCALE / u_ao):** Strength multiplier. Typical values: $0.6$ [S2, unofficial], $3.0$ [S4, unofficial].
- **$D_{max}$ (AO_DISTANCE):** Maximum sampling distance. Typical value: $0.25$ [S3, unofficial].

### Edge Cases and Pitfalls
- **Self-Occlusion:** Sampling too close to the surface ($len_i \approx 0$) can result in $d_i < 0$ (inside the mesh), leading to incorrect large differences. An initial offset $D_{start}$ is required [S2, S3, unofficial].
- **Precision:** Floating-point precision errors in SDF queries can cause flickering or incorrect occlusion in tight crevices. Using quadratic spacing [S2, unofficial] can help distribute samples more effectively over distance.
- **Directionality:** Sampling only along the normal assumes ambient light comes primarily from the hemisphere defined by the normal. For full hemisphere sampling, multiple directions are needed, but SDF-AO typically approximates this with the normal direction for performance [S1, unofficial][S5].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_ambient_occlusion(p: Vector3, n: Vector3, sdf: Function, 
                          num_samples: int = 8, 
                          start_dist: float = 0.01, 
                          step_factor: float = 0.02, 
                          weight_decay: float = 0.85, 
                          strength: float = 0.6) -> float:
    """
    Calculate SDF-based Ambient Occlusion.
    
    Args:
        p: Surface point.
        n: Surface normal.
        sdf: Function returning signed distance to nearest surface.
        num_samples: Number of samples along the normal.
        start_dist: Initial offset distance.
        step_factor: Factor for quadratic spacing (len = start + step * i^2).
        weight_decay: Decay factor for sample weights.
        strength: Multiplier for final occlusion value.
        
    Returns:
        AO factor between 0.0 (fully occluded) and 1.0 (fully lit).
    """
    occ = 0.0
    weight = 1.0
    
    for i in range(num_samples):
        # Calculate sample distance along normal
        # Using quadratic spacing as per [S2, unofficial]
        len_i = start_dist + step_factor * (i * i)
        
        # Sample point
        sample_point = p + n * len_i
        
        # Query SDF
        dist = sdf(sample_point)
        
        # Accumulate occlusion if geometry is closer than sample point
        # max(0.0, len_i - dist) ensures only closer geometry contributes
        occ += max(0.0, len_i - dist) * weight
        
        # Decay weight for next sample
        weight *= weight_decay
        
    # Final AO factor
    # Clamp to [0, 1] and invert (1.0 - occlusion)
    ao = 1.0 - clamp(strength * occ, 0.0, 1.0)
    
    return ao
```

## Subtopics

### Cone Tracing AO
An extension by Wright [S1, unofficial] that replaces the simple normal sampling with cone tracing. Cones originate at the surface and are tested for intersection against the SDF. This provides more accurate occlusion for complex geometry but is more expensive.

### Linear vs. Quadratic Sampling
Linear sampling [S3, S4, unofficial] is simpler but may miss occluders in sparse regions. Quadratic sampling [S2, unofficial] increases sample density further from the surface, better capturing distant occluders for ambient light.

## Operating characteristics
- **Time Complexity:** $O(N)$ per surface point, where $N$ is the number of samples. Independent of scene complexity, depending only on SDF query cost.
- **Memory Complexity:** $O(1)$ per query if SDF is implicit, or $O(V)$ for grid-based SDF storage, where $V$ is the volume of the grid.
- **Performance:** Very fast compared to ray tracing or SSAA with multiple samples. Suitable for real-time applications [S1, unofficial][S5].
- **Strengths:**
  - Geometrically accurate compared to SSAO [S5].
  - No dependence on screen space or depth buffers [S5].
  - Works well for volumetric and implicit surfaces [S1, unofficial].
- **Limitations:**
  - Approximate; may not capture complex indirect lighting bounces [S1, unofficial].
  - Directional bias (normal-only sampling) may look incorrect on flat surfaces or in tight corners [S5].
  - Requires an SDF representation of the scene [S1, unofficial].

## Background
The technique was introduced by Evans [S1, unofficial] as a method to embed objects into a 3D grid of signed distance fields, where occlusion is calculated via a heuristic combining values sampled at increasing distances along the surface normal. It was later expanded by Wright [S1, unofficial] who substituted the ad hoc heuristic with cone tracing. The method is widely used in real-time ray marching and voxel-based rendering engines due to its efficiency and visual quality [S1, unofficial][S5].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S3] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S4] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S5] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
