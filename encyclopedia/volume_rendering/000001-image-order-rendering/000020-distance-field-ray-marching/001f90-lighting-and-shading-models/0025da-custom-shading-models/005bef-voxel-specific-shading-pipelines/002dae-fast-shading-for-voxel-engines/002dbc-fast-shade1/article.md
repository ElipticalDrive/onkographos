# fast_shade1

## Overview
fast_shade1 is a fast shading algorithm for voxel engines that assumes vertical sun rays [S1, unofficial]. It modifies voxel palette indices to simulate lighting, working with palettes where indices 1-32, 33-64, etc., represent dark-to-light gradients [S1, unofficial]. It is faster but less visually detailed than its sibling [fast_shade2](5607-fast-shade2.md), using fewer neighbor checks [S1, unofficial].
## How it works
**Inputs and outputs:**
- Input: A 3D voxel grid of dimensions $xsize \times ysize \times zsize$, where each voxel stores a palette index (integer). Index 0 represents empty space.
- Output: The same voxel grid with modified palette indices to reflect shading.

**Data structures:**
- Voxel grid: A 3D array or linear buffer indexed by $(x, y, z)$.
- Palette index: An integer value. The shading logic relies on the grouping of indices into tone groups of size 32.

**Parameters:**
- $xsize, ysize, zsize$: The dimensions of the current level in voxels [S1, unofficial].

**Steps:**
1. Iterate through the voxel grid. The loops run from 1 to $xsize-2$, 1 to $ysize-2$, and 1 to $zsize-2$ [S1, unofficial]. This boundary exclusion avoids checking neighbors outside the chunk boundaries for the current processing scope [G].
2. Retrieve the current voxel's palette index: $v = \text{get\_voxel\_pal}(x, y, z)$ [S1, unofficial].
3. Calculate the lower bound (color margin) for the current tone group:
   $$ m = \lfloor \frac{v-1}{32} \rfloor \times 32 + 1 $$
   where $\lfloor \cdot \rfloor$ is the floor function [S1, unofficial].
4. If $v \neq 0$ (the voxel is occupied), calculate the shading adjustment by checking specific neighbors:
   - Subtract 3 if the voxel at $(x, y+1, z)$ is occupied (non-zero).
   - Subtract 2 if the voxel at $(x+1, y+1, z)$ is occupied.
   - Subtract 1 if the voxel at $(x+1, y, z+1)$ is occupied OR if the voxel at $(x-1, y, z-1)$ is occupied.
   Let $d$ be the sum of these subtractions.
5. Update the voxel index: $v_{new} = v - d$.
6. Clamp the result to the tone group lower bound: if $v_{new} < m$, set $v_{new} = m$ [S1, unofficial].
7. Write the modified voxel index back: $\text{set\_voxel\_pal}(x, y, z, v_{new})$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_shade1(grid, xsize, ysize, zsize):
    for x in range(1, xsize - 1):
        for y in range(1, ysize - 1):
            for z in range(1, zsize - 1):
                voxel = grid[x, y, z]
                
                # Skip empty voxels
                if voxel == 0:
                    continue
                
                # Calculate tone group lower bound
                color_margin = ((voxel - 1) // 32) * 32 + 1
                
                # Calculate shading discount based on neighbors
                discount = 0
                
                # Check (x, y+1, z): subtract 3 if occupied
                if grid[x, y + 1, z] != 0:
                    discount += 3
                
                # Check (x+1, y+1, z): subtract 2 if occupied
                if grid[x + 1, y + 1, z] != 0:
                    discount += 2
                
                # Check (x+1, y, z+1) OR (x-1, y, z-1): subtract 1 if either occupied
                if grid[x + 1, y, z + 1] != 0 or grid[x - 1, y, z - 1] != 0:
                    discount += 1
                
                # Apply shading
                new_voxel = voxel - discount
                
                # Clamp to tone group lower bound
                if new_voxel < color_margin:
                    new_voxel = color_margin
                
                grid[x, y, z] = new_voxel
```

**Edge cases and pitfalls:**
- The voxel index must not drop below the lower bound of its tone group ($m$). If $v_{new} < m$, clamp it to $m$ [S1, unofficial].
- The iteration bounds exclude the outermost layer of voxels (0 and $size-1$) [S1, unofficial]. This implies the algorithm processes interior voxels of a chunk, likely assuming boundary voxels are handled separately or are static [G].
- The algorithm assumes a specific palette structure where brightness is grouped in blocks of 32 [S1, unofficial].

## Subtopics

### [Voxel Neighborhood Light Gathering](13187-voxel-neighborhood-light-gathering.md)
Gathers the light intensity of the 26 neighbors surrounding a specific voxel within a chunk to determine local lighting conditions for smoothing calculations.

## Operating characteristics
**Time and memory complexity:**
- Time complexity: $O(N)$, where $N$ is the number of voxels processed. Each voxel requires a constant number of neighbor checks (up to 4 specific neighbors) and arithmetic operations.
- Memory complexity: $O(1)$ auxiliary space, as it modifies the grid in place.

**Measured performance:**
- Not given in the sources.

**Scaling:**
- Not given in the sources.

**Strengths and limitations:**
- Strengths: Fast execution due to minimal neighbor checks and simple arithmetic [S1, unofficial]. Suitable for CPU-only rendering engines [S1, unofficial].
- Limitations: Less visually detailed than [fast_shade2](5607-fast-shade2.md) [S1, unofficial]. Assumes vertical sun rays, limiting directional lighting flexibility [S1, unofficial].

**When to use it and when not:**
- Use when performance is critical and vertical lighting is sufficient.
- Do not use when complex directional lighting or higher visual fidelity is required.

## Background
**Origin:**
- The algorithm originates from the DOOMMY voxel engine, an unofficial voxel-based game engine that uses only CPU for rendering and Lua for scripting [S1, unofficial].
- Author: jabberx.
- Date: 2012.

**History and development:**
- The algorithm is part of the DOOMMY engine's shading system, designed to provide basic lighting effects efficiently on CPU hardware [S1, unofficial].

**Where it is used:**
- DOOMMY voxel engine [S1, unofficial].

## Sources
[S1] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
