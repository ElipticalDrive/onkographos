# Fast Shading for Voxel Engines


## Overview
Fast Shading for Voxel Engines is a CPU-based lighting algorithm designed for voxel engines that assumes vertical sun rays to simulate illumination by modifying voxel palette indices directly [S1, unofficial]. It differs from standard radiometric lighting models by operating on discrete tone groups within a palette, where indices 1–32, 33–64, etc., represent dark-to-light gradients, allowing for fast, artifact-free shading without complex ray tracing or normal calculations [S1, unofficial]. Unlike [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) which computes physically based reflections, or [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md) which may involve GPU acceleration, this method is strictly CPU-bound and optimized for immediate visual feedback in simple voxel environments [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid of dimensions $xsize \times ysize \times zsize$, where each voxel stores a palette index.
- Output: The same voxel grid with modified palette indices to reflect shading.

**Data Structures:**
- The voxel grid is accessed via `get_voxel_pal(x, y, z)` and `set_voxel_pal(x, y, z, val)`.
- Palette indices are integers. Empty voxels are typically index 0.
- Indices are grouped into tone groups of 32 (e.g., 1–32, 33–64).

**Algorithm Steps:**
1. Iterate through the voxel grid from $x=1$ to $xsize-2$, $y=1$ to $ysize-2$, and $z=1$ to $zsize-2$. Boundary voxels are skipped to avoid out-of-bounds checks for neighbors [S1, unofficial].
2. Retrieve the current voxel's palette index: `voxel = get_voxel_pal(x, y, z)`.
3. Calculate the lower bound (color margin) for the current tone group:
   $$ color\_margin = \lfloor \frac{voxel - 1}{32} \rfloor \times 32 + 1 $$
   This ensures the shaded color remains within the same tonal band [S1, unofficial].
4. If `voxel != 0` (the voxel is occupied), apply shading by subtracting values based on the occupancy of neighboring voxels:
   - Subtract 3 if the voxel directly above `(x, y+1, z)` is occupied (non-zero).
   - Subtract 2 if any of the following are occupied: `(x+1, y+1, z)`, `(x-1, y+1, z)`, `(x+1, y+1, z+1)`, or `(x+1, y+1, z-1)`.
   - Subtract 1 if any of the following are occupied: `(x+1, y, z+1)`, `(x-1, y, z-1)`, `(x+1, y, z-1)`, or `(x-1, y, z+1)`.
5. Clamp the resulting voxel index to the lower bound: if `voxel < color_margin`, set `voxel = color_margin` [S1, unofficial].
6. Write the modified index back: `set_voxel_pal(x, y, z, voxel)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_shade2(xsize, ysize, zsize, get_voxel_pal, set_voxel_pal):
    for x in range(1, xsize - 1):
        for y in range(1, ysize - 1):
            for z in range(1, zsize - 1):
                voxel = get_voxel_pal(x, y, z)
                
                # Skip empty voxels
                if voxel == 0:
                    continue
                
                # Calculate tone group lower bound
                color_margin = ((voxel - 1) // 32) * 32 + 1
                
                shading = 0
                
                # Check neighbors for shading contribution
                # Subtract 3 if directly above is occupied
                if get_voxel_pal(x, y + 1, z) != 0:
                    shading += 3
                
                # Subtract 2 if specific upper neighbors are occupied
                if (get_voxel_pal(x + 1, y + 1, z) != 0 or
                    get_voxel_pal(x - 1, y + 1, z) != 0 or
                    get_voxel_pal(x + 1, y + 1, z + 1) != 0 or
                    get_voxel_pal(x + 1, y + 1, z - 1) != 0):
                    shading += 2
                
                # Subtract 1 if specific side neighbors are occupied
                if (get_voxel_pal(x + 1, y, z + 1) != 0 or
                    get_voxel_pal(x - 1, y, z - 1) != 0 or
                    get_voxel_pal(x + 1, y, z - 1) != 0 or
                    get_voxel_pal(x - 1, y, z + 1) != 0):
                    shading += 1
                
                # Apply shading and clamp
                voxel = voxel - shading
                if voxel < color_margin:
                    voxel = color_margin
                
                set_voxel_pal(x, y, z, voxel)
```

**Parameters:**
- `xsize`, `ysize`, `zsize`: Dimensions of the voxel level [S1, unofficial].

**Edge Cases and Pitfalls:**
- The voxel index must not drop below the lower bound of its tone group (`color_margin`). Failure to clamp results in invalid palette indices or visual artifacts [S1, unofficial].
- Boundary voxels are excluded from processing to simplify neighbor access [S1, unofficial].

## Subtopics

### fast_shade1
A faster but less visually detailed variant of the shading algorithm, also assuming vertical sun rays [S1, unofficial]. It reduces the number of neighbor checks compared to `fast_shade2`. The shading logic subtracts 3 if `(x, y+1, z)` is occupied, 2 if `(x+1, y+1, z)` is occupied, and 1 if `(x+1, y, z+1)` or `(x-1, y, z-1)` is occupied [S1, unofficial]. It uses the same tone group clamping logic and iteration bounds as `fast_shade2` [S1, unofficial]. Choose this variant when performance is critical and the reduced shading detail is acceptable.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels processed. The constant factor is low due to simple integer arithmetic and neighbor checks.
- **Memory Complexity:** $O(1)$ auxiliary space, as it modifies the grid in place.
- **Performance:** Optimized for CPU execution in voxel engines like DOOMMY, which rely on CPU rendering [S1, unofficial].
- **Strengths:** Extremely fast, no floating-point operations, simple implementation, works well with palette-based rendering.
- **Limitations:** Assumes vertical sun rays, limited shading detail compared to normal-based lighting, only works with specific palette structures (tone groups of 32) [S1, unofficial].
- **When to use:** Simple voxel games, CPU-only rendering pipelines, real-time shading where performance is paramount.
- **When not to use:** Physically based rendering, scenes with complex lighting angles, or when high-fidelity shading is required.

## Background
Fast Shading for Voxel Engines was developed as part of the DOOMMY voxel engine, an unofficial voxel-based game engine that uses only CPU for rendering and Lua for scripting [S1, unofficial]. The engine was created by jabberx and released under the GNU General Public License [S1, unofficial]. The shading method is designed to provide basic lighting effects efficiently on CPU hardware, reflecting the constraints and goals of early voxel engine development [S1, unofficial].

## Sources
[S1] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
