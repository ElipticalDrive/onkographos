# Fast Shading for Voxel Engines


## Overview
Fast Shading for Voxel Engines is a CPU-based lighting algorithm designed for voxel engines that assumes vertical sun rays. It modifies voxel palette indices to simulate lighting by checking immediate neighbors, operating on palettes where indices are grouped into dark-to-light gradients (e.g., 1-32, 33-64) [S1, unofficial]. It is faster but less visually detailed than its sibling [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md), using fewer neighbor checks than more complex models [S1, unofficial].

## How it works
**Inputs:** A 3D voxel grid of dimensions $xsize \times ysize \times zsize$, where each voxel stores a palette index.
**Outputs:** The same voxel grid with modified palette indices representing shaded lighting.

**Data Structures:**
The voxel grid is indexed by integer coordinates $(x, y, z)$. Each voxel contains a single integer representing a palette index. The palette is structured such that indices $1$ to $32$ represent the first tone group, $33$ to $64$ the second, and so on.

**Parameters:**
- $xsize, ysize, zsize$: The dimensions of the voxel grid.

**Algorithm Steps:**
1. Iterate through every voxel in the grid, excluding the boundary layers to avoid out-of-bounds checks. The loops run for $x$ from $1$ to $xsize-2$, $y$ from $1$ to $ysize-2$, and $z$ from $1$ to $zsize-2$.
2. Retrieve the current voxel's palette index: $v = \text{get\_voxel\_pal}(x, y, z)$.
3. If $v = 0$ (empty), skip to the next voxel.
4. Calculate the lower bound (color margin) for the current tone group:
   $$ m = \lfloor \frac{v-1}{32} \rfloor \times 32 + 1 $$
5. Initialize a shading offset $d = 0$.
6. Check neighbors to determine occlusion/shading based on the assumption of vertical sun rays:
   - If the voxel at $(x, y+1, z)$ is occupied ($v_{top} \neq 0$), subtract 3 from the potential brightness (add 3 to offset $d$).
   - If any of the voxels at $(x+1, y+1, z)$, $(x-1, y+1, z)$, $(x+1, y+1, z+1)$, or $(x+1, y+1, z-1)$ are occupied, subtract 2 from the potential brightness (add 2 to offset $d$).
   - If any of the voxels at $(x+1, y, z+1)$, $(x-1, y, z-1)$, $(x+1, y, z-1)$, or $(x-1, y, z+1)$ are occupied, subtract 1 from the potential brightness (add 1 to offset $d$).
   *Note: The source specifies specific neighbor coordinates for each weight. The logic implies that higher $y$ neighbors (above) have a stronger shading effect, consistent with vertical sun rays being blocked by overhead geometry.*
7. Calculate the new index: $v_{new} = v - d$.
8. Clamp the new index to the lower bound of its tone group:
   $$ \text{if } v_{new} < m, \text{ then } v_{new} = m $$
9. Write the modified voxel index back: $\text{set\_voxel\_pal}(x, y, z, v_{new})$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_shade_voxels(grid, xsize, ysize, zsize):
    for x in range(1, xsize - 1):
        for y in range(1, ysize - 1):
            for z in range(1, zsize - 1):
                v = grid[x, y, z]
                
                # Skip empty voxels
                if v == 0:
                    continue
                
                # Calculate tone group lower bound
                color_margin = ((v - 1) // 32) * 32 + 1
                
                shading_offset = 0
                
                # Check neighbors for shading
                # Weight 3: Directly above
                if grid[x, y + 1, z] != 0:
                    shading_offset += 3
                
                # Weight 2: Upper neighbors (diagonal and cardinal in Y+ plane)
                if (grid[x + 1, y + 1, z] != 0 or
                    grid[x - 1, y + 1, z] != 0 or
                    grid[x + 1, y + 1, z + 1] != 0 or
                    grid[x + 1, y + 1, z - 1] != 0):
                    shading_offset += 2
                
                # Weight 1: Side neighbors (diagonal in Y plane)
                if (grid[x + 1, y, z + 1] != 0 or
                    grid[x - 1, y, z - 1] != 0 or
                    grid[x + 1, y, z - 1] != 0 or
                    grid[x - 1, y, z + 1] != 0):
                    shading_offset += 1
                
                # Apply shading
                new_v = v - shading_offset
                
                # Clamp to tone group lower bound
                if new_v < color_margin:
                    new_v = color_margin
                
                grid[x, y, z] = new_v
```

## Subtopics

### [fast_shade1](5608-fast-shade1.md)
fast_shade1 is a fast shading algorithm for voxel engines that assumes vertical sun rays. It modifies voxel palette indices to simulate lighting, working with palettes where indices 1-32, 33-64, etc., represent dark-to-light gradients. It is faster but less visually detailed than its sibling [fast_shade2](5607-fast-shade2.md), using fewer neighbor checks.

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of voxels, as it iterates through the grid once. The constant factor is low due to simple integer arithmetic and neighbor lookups.
**Memory Complexity:** $O(1)$ auxiliary space, operating in-place on the voxel grid.
**Performance:** Designed for CPU-only rendering engines. It is significantly faster than ray-marching or complex normal-based shading because it avoids floating-point lighting calculations and long-distance ray casts.
**Strengths:** Extremely fast, suitable for real-time CPU rendering in voxel engines. Simple to implement.
**Limitations:** Visual quality is limited by the discrete palette steps and the simplified lighting model (vertical sun rays only). It does not support dynamic light sources or complex material properties. The shading is static relative to the voxel structure.

## Background
Fast Shading for Voxel Engines was developed as part of the DOOMMY voxel engine, a voxel-based game engine that uses only CPU for rendering and Lua for scripting. The engine was created by jabberx in 2012 [S1, unofficial]. The method is designed to provide basic lighting effects in a resource-constrained environment without GPU acceleration.

## Sources
[S1] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
