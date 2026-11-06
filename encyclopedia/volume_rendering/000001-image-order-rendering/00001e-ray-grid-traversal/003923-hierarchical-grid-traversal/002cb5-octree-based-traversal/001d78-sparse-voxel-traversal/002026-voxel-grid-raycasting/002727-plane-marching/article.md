# Plane Marching

Also called: Raymarching Plane Intersections

## Overview
Plane Marching is a raymarching technique for voxel grids that advances a ray by stepping to the nearest intersection with the integer grid planes separating voxel layers [S1, unofficial][S2, unofficial]. It solves the problem of traversing a dense voxel volume to find the first solid voxel by checking occupancy at each plane boundary [S1, unofficial][S2, unofficial]. Unlike shape-assisted siblings such as [Sphere Assisted Raymarching](3865-sphere-assisted-raymarching.md) or [Optimised Voxel Ray Marching](4240-optimised-voxel-ray-marching.md), it does not store auxiliary distance fields or shape metadata, relying solely on the geometric structure of the grid [S1, unofficial][S2, unofficial]. This makes it simple to implement but fragile in empty space, as it never skips more than one voxel per step [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $p_0$ (voxel-unit space), ray direction $d$ (voxel-unit space), voxel grid texture `voxels` (3D texture containing color/alpha), `voxelGridSize` (dimensions of the grid in voxels), `maxDistToCheck` (maximum traversal distance) [S1, unofficial].
- Output: Color `vec4` of the first solid voxel encountered, or transparent black if no solid voxel is found within `maxDistToCheck` [S1, unofficial].

**Data Structures:**
- `voxels`: A 3D texture where each texel stores the color and alpha of a voxel [S1, unofficial]. The alpha channel determines solidity ($c.a > 0$) [S1, unofficial]. The texture coordinates are normalized to $[0, 1]$ based on `voxelGridSize` [S1, unofficial].

**Algorithm Steps:**
1. Initialize step distance $t = 0$ [S1, unofficial].
2. Loop while $t \le \text{maxDistToCheck}$:
   a. Calculate current position $p = p_0 + d \cdot t$ [S1, unofficial].
   b. Convert $p$ to texture coordinates by dividing by `voxelGridSize` [S1, unofficial].
   c. If the sampled alpha $c.a > 0$, return $c$ [S1, unofficial].
   d. Calculate the distance to the next integer grid plane along each axis [S1, unofficial].
   e. Update $t$ by adding the smallest positive delta plus a small epsilon [S1, unofficial].

**Formulas:**
The distance to the next plane intersection along each axis is computed using the ray direction $d$ and current position $p$ [S2, unofficial]. In voxel-unit-space (where one voxel is 1x1x1 units), the delta for each axis component is:

$$ \text{deltas}_i = \frac{\text{step}(0, d_i) - \text{fract}(p_i)}{d_i} $$

Where:
- $i \in \{x, y, z\}$
- $\text{step}(0, d_i)$ returns $1$ if $d_i \ge 0$, else $0$ [S1, unofficial]. This effectively computes $\lceil p_i \rceil - p_i$ for positive directions and $\lfloor p_i \rfloor - p_i$ for negative directions [S1, unofficial][S2, unofficial].
- $\text{fract}(p_i)$ is the fractional part of $p_i$ ($p_i - \lfloor p_i \rfloor$) [S1, unofficial].
- $d_i$ is the $i$-th component of the ray direction [S1, unofficial].

The step size to advance is the minimum of these deltas, ensuring the ray moves to the closest plane intersection:

$$ t_{\text{new}} = t + \max(\min(\text{deltas}_x, \text{deltas}_y, \text{deltas}_z), \epsilon) $$

Where $\epsilon$ is a tiny constant to prevent infinite loops due to floating-point precision errors [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def plane_march(p0, d, voxels, voxel_grid_size, max_dist_to_check, epsilon=1e-6):
    # sample_3d_texture is a standard trilinear 3D texture lookup returning vec4
    t = 0.0
    while t <= max_dist_to_check:
        # Calculate current position
        p = p0 + d * t
        
        # Sample voxel color
        tex_coords = p / voxel_grid_size
        c = sample_3d_texture(voxels, tex_coords)
        
        # Check for solidity
        if c.a > 0:
            return c
        
        # Calculate deltas to next plane boundaries
        deltas = []
        for i in range(3):
            step_val = 1.0 if d[i] >= 0 else 0.0
            frac_val = p[i] - math.floor(p[i])
            deltas.append((step_val - frac_val) / d[i])
        
        # Find the minimum positive delta to step to the closest plane
        # Add epsilon to guarantee progress and avoid floating point errors
        t += max(min(deltas), epsilon)
        
    return vec4(0.0, 0.0, 0.0, 0.0)
```

**Parameters:**
- `epsilon`: A small constant (e.g., $10^{-6}$) added to the step size to guarantee progress and prevent infinite loops caused by floating-point inaccuracies [S1, unofficial][S2, unofficial]. The exact value is not specified in the sources but must be "tiny" [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Floating Point Errors:** Accumulation of floating-point errors can cause "tunneling" (skipping voxels) and subtle artifacts [S1, unofficial][S2, unofficial]. The epsilon helps but does not fully eliminate this fragility [S1, unofficial].
- **Degenerate Performance:** In empty space or when the target voxel is far away, the algorithm checks every single voxel along the ray path because it never moves more than one voxel at a time [S1, unofficial][S2, unofficial]. This leads to poor performance in degenerate cases, such as a single solid voxel in the corner of a large empty grid [S2, unofficial].
- **Fillrate Limitation:** The technique is fragment shader heavy and is likely to be fillrate limited on typical hardware [S1, unofficial][S2, unofficial].

## Subtopics

### [Sphere Assisted Raymarching](3865-sphere-assisted-raymarching.md)
Stores radius data in empty voxels to skip large distances. Choose when performance in sparse volumes is critical.

### [Optimised Voxel Ray Marching](4240-optimised-voxel-ray-marching.md)
Optimizes traversal logic with early exits and culling. Choose for better performance in complex scenes.

### [Voxel-Cell Duality Mapping](2240-voxel-cell-duality-mapping.md)
Maps voxels to cells for different traversal strategies. Choose for alternative spatial indexing.

### [Spherical Volume Ray Walking](3865-spherical-volume-ray-walking.md)
Uses spherical stepping for non-axis-aligned volumes. Choose for spherical volume representations.

### [Regular Grid](3950-regular-grid.md)
General regular grid traversal. Choose for non-voxel regular structures.

### [Voxel-based Ray Tracing for GI and Specular Reflections](4091-voxel-based-ray-tracing-for-gi-and-specular-reflections.md)
Extends ray marching for global illumination. Choose for advanced lighting.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels intersected by the ray [S1, unofficial][S2, unofficial]. In the worst case (empty space), $N$ is proportional to the distance to the first solid voxel or the grid boundary [S1, unofficial][S2, unofficial].
- **Memory Complexity:** $O(V)$ where $V$ is the total number of voxels in the grid, as the entire voxel set must be loaded into a 3D texture [S2, unofficial]. Memory usage is data-independent regarding sparsity; a 32x32x32 grid always consumes the same memory regardless of how many voxels are solid [S2, unofficial].
- **Measured Performance:** In a degenerate case (32x32x32 grid with a single solid voxel in the far corner), Plane Marching shows vastly worse performance compared to shape-assisted techniques [S2, unofficial].
- **Strengths:** Performant in parallelized environments (e.g., GPUs), requires little upfront computation, and is straightforward to implement [S1, unofficial][S2, unofficial].
- **Limitations:** Fragile due to floating-point errors, fillrate limited, and inefficient in empty space [S1, unofficial][S2, unofficial].
- **When to Use:** Simple voxel rendering where performance is not critical or the volume is dense [S1, unofficial][S2, unofficial].
- **When Not to Use:** Sparse volumes, large empty spaces, or when high performance is required [S1, unofficial][S2, unofficial].

## Background
Plane Marching is described as one of the most interesting ways to render voxels, performant and parallelizable [S1, unofficial][S2, unofficial]. It is based on the concept of raymarching plane intersections, where a ray is advanced along the planes separating voxel layers [S1, unofficial][S2, unofficial]. The technique is often compared to Inigo Quilez's ray marching work, though Quilez's work primarily focuses on signed distance fields, whereas Plane Marching uses unsigned distance fields (grid planes) [S1, unofficial][S2, unofficial]. The method was popularized in the context of voxel rendering for real-time applications, such as Minecraft-like games, by developers like Caleb Leak [S1, unofficial][S2, unofficial].

## Sources
[S1] Raymarching Voxel Rendering | Caleb Leak, https://calebleak.com/posts/raymarching-voxel-rendering/, fetched 2026-10-24 (unofficial)
[S2] Raymarching Voxel Rendering, https://medium.com/@calebleak/raymarching-voxel-rendering-58018201d9d6, fetched 2026-10-25 (unofficial)
