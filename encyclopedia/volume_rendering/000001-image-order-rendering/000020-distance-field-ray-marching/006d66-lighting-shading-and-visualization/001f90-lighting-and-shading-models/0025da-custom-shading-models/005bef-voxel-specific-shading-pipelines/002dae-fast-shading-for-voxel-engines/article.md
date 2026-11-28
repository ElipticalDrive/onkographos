# Fast Shading for Voxel Engines

## Overview
Fast Shading for Voxel Engines is a CPU-based lighting algorithm for voxel engines that assumes vertical sun rays [S1, unofficial]. It modifies voxel palette indices to simulate lighting by checking immediate neighbors, operating on palettes where indices 1–32, 33–64, etc., represent dark-to-light gradients [S1, unofficial]. It is faster but less visually detailed than its sibling [fast_shade1](5608-fast-shade1.md), using fewer neighbor checks [S1, unofficial]. Compared to [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md), [Voxel Shader Volumetric Ray Marching](16930-voxel-shader-volumetric-ray-marching.md), and [Voxmat Path Tracer](21082-voxmat-path-tracer.md), it avoids ray casting and floating-point math, trading accuracy for CPU-only speed [G].

## How it works
**Inputs:** A 3D voxel grid of dimensions $xsize \times ysize \times zsize$, where each voxel stores a palette index [S1, unofficial].
**Outputs:** The same voxel grid with modified palette indices representing shaded lighting [S1, unofficial].

**Data Structures:**
The voxel grid is indexed by integer coordinates $(x, y, z)$ [S1, unofficial]. Each voxel contains a single integer representing a palette index [S1, unofficial]. The palette is structured such that indices $1$ to $32$ represent the first tone group, $33$ to $64$ the second, and so on [S1, unofficial].

**Parameters:**
- $xsize, ysize, zsize$: The dimensions of the current level in voxels [S1, unofficial].

**Algorithm Steps:**
1. Iterate $x$ from $1$ to $xsize-2$, $y$ from $1$ to $ysize-2$, and $z$ from $1$ to $zsize-2$ [S1, unofficial].
2. Retrieve the current voxel's palette index: $v = \text{get\_voxel\_pal}(x, y, z)$ [S1, unofficial].
3. If $v = 0$ (empty), skip to the next voxel [S1, unofficial].
4. Initialize a shading offset $d = 0$ [S1, unofficial].
5. Check neighbors to determine occlusion/shading based on the assumption of vertical sun rays [S1, unofficial]:
   - If the voxel at $(x, y+1, z)$ is occupied ($v_{top} \neq 0$), subtract 3 from the potential brightness (add 3 to offset $d$) [S1, unofficial].
   - If any of the voxels at $(x+1, y+1, z)$, $(x-1, y+1, z)$, $(x+1, y+1, z+1)$, or $(x+1, y+1, z-1)$ are occupied, subtract 2 from the potential brightness (add 2 to offset $d$) [S1, unofficial].
   - If any of the voxels at $(x+1, y, z+1)$, $(x-1, y, z-1)$, $(x+1, y, z-1)$, or $(x-1, y, z+1)$ are occupied, subtract 1 from the potential brightness (add 1 to offset $d$) [S1, unofficial].
6. Calculate the new index: $v_{new} = v - d$ [S1, unofficial].
7. Clamp the new index to the lower bound of its tone group: if $v_{new} < m$, then $v_{new} = m$ [S1, unofficial].
8. Write the modified voxel index back: $\text{set\_voxel\_pal}(x, y, z, v_{new})$ [S1, unofficial].

**Formulas:**
- Tone group lower bound (color margin): $m = \lfloor (v-1)/32 \rfloor \times 32 + 1$, where $v$ is the current palette index and $\lfloor \cdot \rfloor$ denotes the floor function [S1, unofficial].
- Shading offset: $d = 3 \cdot \mathbb{I}(v_{x,y+1,z} \neq 0) + 2 \cdot \mathbb{I}(\text{any of } v_{x\pm1,y+1,z}, v_{x+1,y+1,z\pm1} \neq 0) + 1 \cdot \mathbb{I}(\text{any of } v_{x\pm1,y,z\pm1} \neq 0)$, where $\mathbb{I}(\cdot)$ is the indicator function [S1, unofficial].

**Edge Cases and Pitfalls:**
- The voxel index must not drop below the lower bound of its tone group ($m$) [S1, unofficial]. If $v_{new} < m$, clamp it to $m$ [S1, unofficial].
- Boundary voxels are excluded from processing to avoid out-of-bounds memory access [S1, unofficial].

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
                
                # Weight 3: Directly above
                if grid[x, y + 1, z] != 0:
                    shading_offset += 3
                
                # Weight 2: Upper neighbors (specific coordinates per source)
                if (grid[x + 1, y + 1, z] != 0 or
                    grid[x - 1, y + 1, z] != 0 or
                    grid[x + 1, y + 1, z + 1] != 0 or
                    grid[x + 1, y + 1, z - 1] != 0):
                    shading_offset += 2
                
                # Weight 1: Side neighbors (specific coordinates per source)
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
**Time Complexity:** $O(N)$ where $N$ is the number of voxels, as it iterates through the grid once [G]. The constant factor is low due to simple integer arithmetic and neighbor lookups [G].
**Memory Complexity:** $O(1)$ auxiliary space, operating in-place on the voxel grid [G].
**Performance:** Designed for CPU-only rendering engines [S1, unofficial]. It is significantly faster than ray-marching or complex normal-based shading because it avoids floating-point lighting calculations and long-distance ray casts [G].
**Strengths:** Extremely fast, suitable for real-time CPU rendering in voxel engines [G]. Simple to implement [G].
**Limitations:** Visual quality is limited by the discrete palette steps and the simplified lighting model (vertical sun rays only) [G]. It does not support dynamic light sources or complex material properties [G]. The shading is static relative to the voxel structure [G].

## Background
Fast Shading for Voxel Engines was developed as part of the DOOMMY voxel engine, a voxel-based game engine that uses only CPU for rendering and Lua for scripting [S1, unofficial]. The engine was created by jabberx in 2012 [S1, unofficial]. The method is designed to provide basic lighting effects in a resource-constrained environment without GPU acceleration [G].

## Sources
[S1] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
