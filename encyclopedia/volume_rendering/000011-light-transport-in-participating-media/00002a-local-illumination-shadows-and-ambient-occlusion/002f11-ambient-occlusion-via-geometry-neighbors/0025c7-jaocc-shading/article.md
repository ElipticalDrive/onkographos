# JAOCC Shading
Also called: Jaocc Occlusion

## Overview
JAOCC Shading is a voxel-based ambient occlusion technique that computes a depth-weighted dimming factor by counting non-empty neighbors in a cubic neighborhood [S3, unofficial]. It addresses hard, physically implausible lighting transitions at voxel boundaries by dynamically computing the normal of the cut voxel face from the local neighborhood geometry instead of using fixed AABB edge normals [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D occupancy grid where each voxel has an associated RGB color and a boolean occupancy status [S3, unofficial].
- Output: A shaded voxel array where RGB values are modulated by an ambient occlusion factor [S3, unofficial].
- Parameters:
  - `count`: The radius of the cubic neighborhood to sample (integer). Determines the sampling box size from $-count$ to $+count$ [S3, unofficial].
  - `depth`: A float parameter controlling the occlusion intensity, typically in the range $[0.01, 1.0]$ [S3, unofficial].

**Data Structures:**
- The voxel grid is stored as a 3D array indexed by $(x, y, z)$ [G].
- Each voxel contains:
  - `occupied`: Boolean flag indicating if the voxel is non-empty [S3, unofficial].
  - `color`: RGB tuple or vector [S3, unofficial].

**Algorithm Steps:**
1. **Precompute Normalization Constant (`max_dim`):**
   Calculate the maximum possible weighted neighbor count for a fully occupied neighborhood [S3, unofficial]. Iterate $xn$ from $-count$ to $count$, $yn$ from $-count$ to $count$, and $zn$ from $-count$ to $count$ [S3, unofficial]. For each neighbor offset $(xn, yn, zn)$, let $yk1 = 1$ if $yn > 0$ else $0$, and $yk2 = 1$ if $yn < 0$ else $0$, then add $(1 + yk1 - yk2)$ to `max_dim` [S3, unofficial]. This weight depends only on the Y-coordinate: neighbors above ($yn>0$) contribute 2, neighbors below ($yn<0$) contribute 0, and neighbors at the same level ($yn=0$) contribute 1 [S3, unofficial].
2. **Iterate Over Voxels:**
   Loop through all voxels in the volume. The iteration range for $x$ is $[count, xsize - count - 1]$, for $y$ is $[count, ysize - count - 1]$, and for $z$ is $[count, zsize - count - 1]$ [S3, unofficial]. Skip voxels that are empty (`occupied == false`) [S3, unofficial].
3. **Compute Occlusion Dimension (`dim`):**
   Initialize `dim` to 0 [S3, unofficial]. Iterate $xn$ from $-count$ to $count$, $yn$ from $-count$ to $count$, and $zn$ from $-count$ to $count$ [S3, unofficial]. Check the occupancy status of the neighbor at $(x+xn, y+yn, z+zn)$ [S3, unofficial]. If the neighbor is occupied, let $yk1 = 1$ if $yn > 0$ else $0$, and $yk2 = 1$ if $yn < 0$ else $0$, then add $(1 + yk1 - yk2)$ to `dim` [S3, unofficial].
4. **Calculate Shading Factor (`dim_factor`):**
   Normalize the computed `dim` by `max_dim` and apply the `depth` parameter [S3, unofficial].
   $$ \text{dim\_factor} = \text{clamp}\left( \left(1.0 - \frac{\text{clamp}(\text{dim}, 0, \text{max\_dim})}{\text{max\_dim}}\right) \cdot \text{depth} + (1 - \text{depth}), 0, 1 \right) $$
   When `dim` is 0 (no neighbors), `dim_factor` is 1.0 (fully lit) [S3, unofficial]. When `dim` is `max_dim` (fully surrounded), `dim_factor` is `1 - depth` [S3, unofficial].
5. **Apply Shading:**
   Multiply the voxel's RGB components by `dim_factor` and clamp the result to $[0, 255]$ [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jaocc_shade(voxel_grid, count, depth):
    """
    voxel_grid: 3D array of objects with attributes .occupied (bool) and .color (tuple of 3 ints 0-255)
    count: int, radius of neighborhood
    depth: float, occlusion intensity [0.01, 1.0]
    """
    xsize, ysize, zsize = voxel_grid.shape
    
    # Step 1: Precompute max_dim
    max_dim = 0
    for xn in range(-count, count + 1):
        for yn in range(-count, count + 1):
            for zn in range(-count, count + 1):
                yk1 = 1 if yn > 0 else 0
                yk2 = 1 if yn < 0 else 0
                max_dim += (1 + yk1 - yk2)
    
    # Step 2: Iterate and shade
    for x in range(count, xsize - count):
        for y in range(count, ysize - count):
            for z in range(count, zsize - count):
                voxel = voxel_grid[x, y, z]
                
                # Skip empty voxels
                if not voxel.occupied:
                    continue
                
                # Step 3: Compute dim
                dim = 0
                for xn in range(-count, count + 1):
                    for yn in range(-count, count + 1):
                        for zn in range(-count, count + 1):
                            neighbor = voxel_grid[x+xn, y+yn, z+zn]
                            if neighbor.occupied:
                                yk1 = 1 if yn > 0 else 0
                                yk2 = 1 if yn < 0 else 0
                                dim += (1 + yk1 - yk2)
                
                # Step 4: Calculate dim_factor
                clamped_dim = max(0, min(dim, max_dim))
                normalized_dim = clamped_dim / max_dim
                dim_factor = (1.0 - normalized_dim) * depth + (1.0 - depth)
                dim_factor = max(0.0, min(1.0, dim_factor))
                
                # Step 5: Apply shading
                r, g, b = voxel.color
                voxel.color = (
                    max(0, min(255, int(r * dim_factor))),
                    max(0, min(255, int(g * dim_factor))),
                    max(0, min(255, int(b * dim_factor)))
                )
```

**Edge Cases and Pitfalls:**
- **Thin Objects:** The algorithm produces errors with thin objects because the neighborhood sampling may not accurately represent the local geometry [S3, unofficial].
- **Boundary Conditions:** The iteration skips voxels within `count` distance of the boundary [S3, unofficial]. Voxels near the edge are not shaded or require special handling, which is not defined in the source [S3, unofficial].
- **Performance:** The method is computationally expensive due to the nested loops over the neighborhood for every voxel [S3, unofficial].

## Subtopics

### Compensation Map
A nonlinear rescaling of computed AO for highly occluded regions to mitigate error from artificial multiple occlusion (objects thinner than $\delta$ in proximity closer than $\delta$) [S4, unofficial]. It is encoded as a 1D luminance texture on the GPU and automatically clamps input AO to $[0, 1]$ [S4, unofficial]. The curve is fitted from AOV results of the Suburb and House scenes to ray traced results, with the right end based on the intuition that a corner with three walls at a right angle has $\sim 75\%$ occlusion [S4, unofficial]. Unlike the rest of the algorithm, it has no physical basis and is an ad hoc solution for scenes violating the $\delta$-thickness assumption, where AO is strictly over-estimated and correction is roughly parabolic [S4, unofficial]. The performance impact of the additional texture fetch is below the threshold of measurement [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot (2 \cdot count + 1)^3)$, where $N$ is the number of occupied voxels in the interior of the volume [G]. The cubic neighborhood size grows cubically with `count` [S3, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space if shading is done in-place [G].
- **Measured Performance:** Described as "slow" in the source [S3, unofficial]. No specific hardware or resolution metrics are provided [S3, unofficial].
- **Strengths:**
  - Produces smooth results for higher neighbor counts [S3, unofficial].
  - Avoids hard lighting transitions at voxel edges by using normals derived from local neighborhood geometry rather than fixed AABB normals [S1, unofficial].
  - Simple to implement as it relies only on occupancy checks [G].
- **Limitations:**
  - Slow performance for large volumes or large `count` values [S3, unofficial].
  - Inaccurate for thin structures [S3, unofficial].
  - Directional bias along the Y-axis due to the weighting scheme [S3, unofficial].
- **When to use:** Suitable for voxel-based environments where surface normals are not available or expensive to compute, and where a simple ambient occlusion effect is acceptable [G].
- **When not to use:** For high-fidelity rendering requiring accurate shadowing, or for scenes with thin geometry [S3, unofficial].

## Background
JAOCC Shading originates from voxel rendering techniques aimed at improving the visual quality of voxel landscapes [S1, unofficial]. The primary motivation was to overcome the limitations of Axis-Aligned Bounding Box (AABB) normals, which cause hard, physically implausible lighting effects where light rays strike voxel faces perpendicularly versus grazing angles [S1, unofficial]. The method was implemented in open-source voxel projects, such as the `doommy` repository, which provided a concrete implementation of the algorithm [S3, unofficial]. It is part of a broader family of local illumination techniques for participating media and voxel grids [G].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S3] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
[S4] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
