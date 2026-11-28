# Slice Artifact Reduction

Also called: Initial rays’ start positions randomization, Ray Start Position Randomization, Ray start jittering, Ray start jittering with solidity check, Wood grain artifact reduction

## Overview
Slice Artifact Reduction is a stochastic sampling technique used in direct volume ray casting to mitigate "wood grain" artifacts—visible slice boundaries caused by regular ray start positions and discrete voxel grids [S1, unofficial][S2, unofficial]. It differs from standard ray casting by randomizing the initial ray origin along the viewing direction and performing a localized "solidity check" to verify that a sampled voxel is truly part of the volume rather than an edge artifact [S1, unofficial][S2, unofficial]. This method effectively breaks the regularity of sampling patterns that cause visible artifacts, regardless of the ray step size relative to voxel dimensions [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset, transfer function parameters (Threshold), ray start position $P_{start}$, ray direction vector $\vec{d}$, step length $L$, noise texture $N$, fragment coordinates $(x, y)$, and far plane distance [S1, unofficial][S2, unofficial].
*   **Output:** Final accumulated color $dst$ for the pixel.

**Data Structures:**
*   **Noise Texture:** A 2D texture $N$ of size $256 \times 256$ containing random values in $[0, 1]$ [S1, unofficial].
*   **Volume Data:** Accessible via a sampling function returning scalar density $V$ at a given position [S1, unofficial].

**Parameters:**
*   **Threshold:** Density threshold for voxel solidity. Typical value: Not given in the sources.
*   **StepLength ($L$):** Distance between sampling points. Typical value: Not given in the sources.
*   **Noise Scale:** Division factor for fragment coordinates to sample the noise texture. Typical value: 256 [S1, unofficial].

**Algorithm Steps:**
1.  **Jitter Ray Start:** Calculate a random offset using the noise texture sampled at a scaled fragment coordinate. Add this offset to the ray start position along the view direction.
    $$ P_{start}' = P_{start} + \vec{s} \cdot N(x/256, y/256) $$
    where $\vec{s}$ is the step vector and $N$ is the noise value [S1, unofficial].
2.  **Initialize Ray:** Set current position $P = P_{start}'$, remaining ray length $R = R_{total}$, and accumulated color $dst = (0, 0, 0, 0)$ [S1, unofficial].
3.  **Ray Casting Loop:** While $dst.a < 1$ and $R > 0$:
    a.  Sample density $V = \text{lookup}(P)$ [S1, unofficial].
    b.  If $V > \text{Threshold}$:
        i.  **Solidity Check:** Perform a localized check to ensure the voxel is solid.
            *   Move back by half-step: $P \leftarrow P - \vec{s} \cdot 0.5$, $V \leftarrow \text{lookup}(P)$ [S1, unofficial].
            *   If $V > \text{Threshold}$, scale offset forward by quarter-step: $\vec{s}_{offset} \leftarrow \vec{s}_{offset} \cdot 0.5$. Else, scale offset backward by quarter-step: $\vec{s}_{offset} \leftarrow \vec{s}_{offset} \cdot (-0.5)$ [S1, unofficial].
            *   Update position and sample: $P \leftarrow P + \vec{s}_{offset}$, $V \leftarrow \text{lookup}(P)$ [S1, unofficial].
        ii. If $V > \text{Threshold}$, proceed with lighting calculations and accumulate color. Break loop if opaque [S1, unofficial].
    c.  Advance ray: $P \leftarrow P + \vec{s}$, $R \leftarrow R - L$ [S1, unofficial].
4.  **Temporal Accumulation (Optional):** If the viewpoint is static, accumulate multiple frames over time to average out the stochastic noise [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slice_artifact_reduction(ray_origin, ray_direction, step_length, threshold, noise_texture, frag_coord, far_plane_dist):
    # Compute step vector from direction and length
    step_vector = ray_direction * step_length
    
    # Jitter ray start position along view direction
    noise_val = noise_texture.sample(frag_coord / 256.0)
    ray_origin += step_vector * noise_val
    
    pos = ray_origin
    ray_length = far_plane_dist
    dst = vec4(0.0)
    
    # Standard volume sampling: returns density at pos [G]
    def lookup(p):
        return volume_data.sample(p)
        
    while dst.a < 1.0 and ray_length > 0:
        V = lookup(pos)
        
        if V > threshold:
            # Solidity check
            s = -step_vector * 0.5
            pos += s
            V = lookup(pos)
            
            if V > threshold:
                s *= 0.5
            else:
                s *= -0.5
            
            pos += s
            V = lookup(pos)
            
            if V > threshold:
                # Lighting calculations (gradient estimation, diffuse lighting)
                # src = vec4(color, 1.0)
                # dst = (1.0 - dst.a) * src + dst
                break
                
        pos += step_vector
        ray_length -= step_length
        
    return dst
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors may cause the solidity check to fail incorrectly if the step size is very small relative to the voxel size [S1, unofficial].
*   **Boundaries:** Ensure the ray start jitter does not push the ray outside the volume bounding box [S2, unofficial].

## Subtopics

### [Ray Start Position Jittering](7198-ray-start-position-jittering.md)
A specific variant focusing solely on randomizing the initial ray origin to break regularity, effective for low step counts or nearest-neighbor interpolation.

### [Temporal Filtering and Jittering](2762-temporal-filtering-and-jittering.md)
Uses temporal accumulation to average out the noise introduced by stochastic jittering, requiring a static viewpoint.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of sampling steps per ray. The solidity check adds a constant overhead (2-3 extra samples) per voxel hit [S1, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory per ray, plus the noise texture storage [S1, unofficial].
*   **Measured Performance:** Reduces visible artifacts significantly without requiring a reduction in step size, thus maintaining performance compared to high-resolution sampling [S1, unofficial].
*   **Strengths:** Effective against "wood grain" artifacts; simple to implement; works well with low step counts [S1, unofficial][S2, unofficial].
*   **Limitations:** Introduces stochastic noise, requiring temporal accumulation for a clean image if the viewpoint is static [S2, unofficial].
*   **When to Use:** When regular sampling patterns cause visible artifacts, such as with nearest-neighbor interpolation or limited step counts [S1, unofficial][S2, unofficial].
*   **When Not to Use:** When high performance is critical and artifacts are acceptable, or when using high-resolution sampling with trilinear/tricubic interpolation [S1, unofficial][S2, unofficial].

## Background
The technique was popularized in the context of GPU-based volume rendering, specifically in tutorials and implementations such as "Volumetric Splatting at The Little Grasshopper" [S1, unofficial]. It addresses a common issue in early volume renderers where regular sampling patterns led to visible slice boundaries [S1, unofficial]. The concept of randomizing ray start positions to break regularity was also discussed in academic literature on GPU volume rendering [S2, unofficial].

## Sources
[S1] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
[S2] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
