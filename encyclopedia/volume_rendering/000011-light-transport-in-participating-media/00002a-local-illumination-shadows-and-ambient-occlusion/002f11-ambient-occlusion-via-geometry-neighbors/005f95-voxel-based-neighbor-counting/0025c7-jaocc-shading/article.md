# JAOCC Shading

Also called: Jaocc Occlusion

## Overview
JAOCC Shading is a voxel-based ambient occlusion technique that computes a dimming factor by counting non-empty neighbors in a cubic neighborhood, applying a depth-weighted correction to the voxel's RGB color [S3, unofficial]. It addresses the visual artifact of hard, physically implausible lighting transitions at Axis-Aligned Bounding Box (AABB) voxel edges by dynamically deriving surface normals from local geometry rather than using fixed face normals [S1, unofficial]. Unlike its siblings, it explicitly incorporates a "depth" parameter to control shading intensity and uses a specific weighted neighbor sum to approximate occlusion, though it is noted for being computationally slow and prone to errors on thin objects [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid representing occupancy (boolean or integer density) and color (RGB). Parameters: `count` (neighborhood radius), `depth` (shading intensity, $0.01 \le depth \le 1.0$) [S3, unofficial].
*   **Output:** A shaded voxel grid where each voxel's color is multiplied by an occlusion factor $AO \in [0, 1]$ [S3, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D array. The implementation in [S3, unofficial] uses a flattened 1D array with indexing `x * ysize + z * ysize * xsize + y` for a grid of size `xsize` x `ysize` x `zsize`.
*   **Neighbor Offsets:** Defined by a cubic radius `count`.

**Algorithm Steps:**

1.  **Iterate Voxels:** Loop through all voxels $(x, y, z)$ in the grid. Skip empty voxels. The loop bounds are typically `count` to `size - count - 1` to avoid boundary checks, or boundary conditions must be handled [S3, unofficial].
2.  **Compute Max Dimension (`max_dim`):** Calculate the maximum possible weighted neighbor count for normalization. This represents the occlusion if all neighbors on one side of the query voxel were solid.
    *   Initialize `max_dim = 0`.
    *   Iterate offsets $(xn, yn, zn)$ from $-count$ to $count$.
    *   For each offset, calculate weight contribution:
        *   If $yn > 0$, add 1.
        *   If $yn < 0$, subtract 1 (effectively removing from the "solid" count for the denominator logic in the source, though the source description is slightly ambiguous, it sums these values).
        *   Always add 1.
        *   *Correction based on [S3, unofficial] pseudocode logic:* The source calculates `max_dim` by iterating all neighbors and summing `1 + (1 if yn>0 else 0) - (1 if yn<0 else 0)`. This effectively weights neighbors based on their Y-position relative to the center, likely to approximate the surface area or solid angle contribution in a simplified manner.
3.  **Compute Occlusion Dimension (`dim`):** Calculate the actual weighted solid neighbor count.
    *   Initialize `dim = 0`.
    *   Iterate offsets $(xn, yn, zn)$ from $-count$ to $count$.
    *   Check if the neighbor at $(x+xn, y+yn, z+zn)$ is non-empty (solid).
    *   If solid, add weight to `dim`:
        *   $yk1 = 1$ if $yn > 0$ else $0$.
        *   $yk2 = 1$ if $yn < 0$ else $0$.
        *   Add $(1 + yk1 - yk2)$ to `dim`.
4.  **Calculate Occlusion Factor (`dim_factor`):**
    *   Normalize the dimension: $norm = \frac{\min(\max(dim, 0), max\_dim)}{max\_dim}$.
    *   Apply depth weighting:
        $$ dim\_factor = \text{clamp}\left( (1.0 - norm) \times depth + (1.0 - depth), 0.0, 1.0 \right) $$
    *   *Note:* When $depth=1$, $dim\_factor = 1.0 - norm$. When $depth=0$, $dim\_factor = 1.0$ (no shading) [S3, unofficial].
5.  **Apply Shading:**
    *   Multiply the voxel's RGB components by `dim_factor`.
    *   Clamp the result to $[0, 255]$ (for 8-bit) or $[0, 1]$ (for float) [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jaocc_shade(voxel_grid, xsize, ysize, zsize, count, depth):
    """
    voxel_grid: 3D array or flattened 1D array of voxels.
    Each voxel has properties: is_solid (bool), r, g, b.
    Returns: Shaded voxel grid.
    """
    # Helper to check if a voxel is solid
    def is_solid(x, y, z):
        if not (0 <= x < xsize and 0 <= y < ysize and 0 <= z < zsize):
            return False
        return voxel_grid[x, y, z].is_solid

    # Precompute max_dim
    max_dim = 0
    for xn in range(-count, count + 1):
        for yn in range(-count, count + 1):
            for zn in range(-count, count + 1):
                # Weight calculation as per [S3, unofficial]
                yk1 = 1 if yn > 0 else 0
                yk2 = 1 if yn < 0 else 0
                weight = 1 + yk1 - yk2
                max_dim += weight

    # If max_dim is 0 (e.g., count=0), avoid division by zero
    if max_dim == 0:
        return voxel_grid

    # Iterate and shade
    for x in range(count, xsize - count):
        for y in range(count, ysize - count):
            for z in range(count, zsize - count):
                if not is_solid(x, y, z):
                    continue

                dim = 0
                for xn in range(-count, count + 1):
                    for yn in range(-count, count + 1):
                        for zn in range(-count, count + 1):
                            if is_solid(x + xn, y + yn, z + zn):
                                yk1 = 1 if yn > 0 else 0
                                yk2 = 1 if yn < 0 else 0
                                weight = 1 + yk1 - yk2
                                dim += weight

                # Normalize
                norm = dim / max_dim
                
                # Apply depth factor
                # Formula: clamp((1.0 - norm) * depth + (1 - depth), 0, 1)
                # Simplified: 1 - norm * depth
                dim_factor = 1.0 - (norm * depth)
                
                # Clamp to [0, 1]
                if dim_factor < 0.0:
                    dim_factor = 0.0
                if dim_factor > 1.0:
                    dim_factor = 1.0

                # Apply to color
                voxel = voxel_grid[x, y, z]
                voxel.r = clamp(voxel.r * dim_factor, 0, 255)
                voxel.g = clamp(voxel.g * dim_factor, 0, 255)
                voxel.b = clamp(voxel.b * dim_factor, 0, 255)

    return voxel_grid

def clamp(value, min_val, max_val):
    if value < min_val: return min_val
    if value > max_val: return max_val
    return value
```

**Edge Cases and Pitfalls:**
*   **Thin Objects:** The algorithm produces errors with thin objects because the neighborhood sampling may not accurately represent the surface curvature or occlusion for structures thinner than the sampling radius [S3, unofficial].
*   **Performance:** The method is slow due to the triple nested loops for neighbor sampling for every voxel [S3, unofficial].
*   **Boundary Conditions:** Voxels near the grid boundary require special handling (skipping or padding) to avoid out-of-bounds access [S3, unofficial].

## Subtopics

### Compensation Map
A nonlinear rescaling technique applied to JAOCC results to mitigate "artificial multiple occlusion" errors [S4, unofficial].
*   **Definition:** A 1D luminance texture (curve) applied to the computed AO factor. It rescales highly occluded regions to correct over-darkening [S4, unofficial].
*   **Mechanism:** The curve is fitted to ray-traced reference data from specific scenes (e.g., Suburb, House). It automatically clamps input AO to $[0, 1]$. The right end of the curve is set based on the intuition that a corner with three walls at right angles has ~75% occlusion [S4, unofficial].
*   **When to Choose:** Use when rendering scenes with objects thinner than the voxel grid resolution $\delta$ in proximity closer than $\delta$, which causes artificial multiple occlusion (overdarkening of adjacent faces) [S4, unofficial].
*   **Comparison:** Unlike the base JAOCC algorithm, the compensation map has no physical basis; it is an ad-hoc correction for scenes violating the $\delta$-thickness assumption. The error it corrects is strictly over-estimated AO, typically worse in highly occluded areas [S4, unofficial].
*   **Performance:** The additional texture fetch cost is below the threshold of measurement [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot (2 \cdot count + 1)^3)$, where $N$ is the number of solid voxels. The cubic neighborhood sampling makes it significantly slower than simple neighbor counting [S3, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space per voxel if computed in-place, or $O(N)$ for the output grid.
*   **Strengths:** Produces smooth shading results for higher neighbor counts; avoids hard AABB edge lighting artifacts by using local geometry [S1, unofficial][S3, unofficial]. Robust to tessellation changes [S4, unofficial].
*   **Limitations:** Slow performance; inaccurate for thin objects; the "depth" parameter is empirical [S3, unofficial].
*   **When to Use:** When visual smoothness is prioritized over performance and objects are sufficiently thick relative to the voxel grid.
*   **When Not to Use:** For real-time applications with large voxel grids or scenes with thin geometry (e.g., fences, wires) without the compensation map.

## Background
JAOCC Shading originates from voxel rendering implementations in open-source projects and academic theses focusing on voxel-based lighting [S1, unofficial][S3, unofficial]. The specific weighted neighbor counting method is documented in the `doommy` project [S3, unofficial]. The concept of using local neighborhood to determine normals and avoid AABB artifacts is discussed in a thesis on raycasting-based voxel rendering [S1, unofficial]. The "Compensation Map" variant was introduced by McGuire in "Ambient Occlusion Volumes" (2010) to address physical inaccuracies in voxel AO [S4, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S3] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
[S4] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
