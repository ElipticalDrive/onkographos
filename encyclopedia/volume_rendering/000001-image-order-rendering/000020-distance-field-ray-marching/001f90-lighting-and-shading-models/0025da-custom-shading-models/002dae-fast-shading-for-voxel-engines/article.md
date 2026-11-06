# Fast Shading for Voxel Engines


## Overview
Fast Shading for Voxel Engines is a CPU-based lighting algorithm designed for voxel engines that assumes vertical sun rays [S1, unofficial]. It modifies voxel palette indices to simulate lighting by darkening voxels based on the occupancy of their immediate neighbors, effectively simulating shadows cast by adjacent blocks [S1, unofficial]. It sets itself apart from general-purpose shading models by operating directly on discrete palette indices rather than computing continuous radiance, and by relying on a fixed, pre-defined neighborhood kernel rather than ray-marching for occlusion [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid of dimensions $x_{size} \times y_{size} \times z_{size}$, where each voxel stores a palette index (integer).
- Output: The same voxel grid with updated palette indices representing shaded colors.

**Data Structures:**
- The voxel grid is typically stored as a 1D array or 3D array of integers (palette indices).
- Palette indices are grouped into tone groups of 32. Indices 1-32 represent the first gradient, 33-64 the second, etc. Index 0 represents empty space.

**Parameters:**
- $x_{size}, y_{size}, z_{size}$: The dimensions of the current voxel level [S1, unofficial].

**Algorithm Steps:**
1. Iterate through the voxel grid. The iteration bounds exclude the outermost layer to prevent boundary checks during neighbor access:
   $$ x \in [1, x_{size}-2] $$
   $$ y \in [1, y_{size}-2] $$
   $$ z \in [1, z_{size}-2] $$
2. For each voxel at $(x, y, z)$, retrieve its current palette index:
   $$ v = \text{get\_voxel\_pal}(x, y, z) $$
3. If $v = 0$ (empty), skip to the next voxel.
4. Calculate the lower bound (color margin) of the current tone group for $v$:
   $$ \text{color\_margin} = \left\lfloor \frac{v-1}{32} \right\rfloor \times 32 + 1 $$
5. Initialize a shading offset $d = 0$.
6. Check neighbors and subtract from the potential brightness (add to $d$) based on occupancy:
   - If neighbor $(x, y+1, z)$ is occupied ($v_{neighbor} \neq 0$), add 3 to $d$.
   - If neighbor $(x+1, y+1, z)$ is occupied, add 2 to $d$.
   - If neighbor $(x-1, y+1, z)$ is occupied, add 2 to $d$.
   - If neighbor $(x+1, y+1, z+1)$ is occupied, add 2 to $d$.
   - If neighbor $(x+1, y+1, z-1)$ is occupied, add 2 to $d$.
   - If neighbor $(x+1, y, z+1)$ is occupied, add 1 to $d$.
   - If neighbor $(x-1, y, z-1)$ is occupied, add 1 to $d$.
   - If neighbor $(x+1, y, z-1)$ is occupied, add 1 to $d$.
   - If neighbor $(x-1, y, z+1)$ is occupied, add 1 to $d$.
   *(Note: The source lists specific neighbors for each weight. The list above aggregates the unique neighbors mentioned in the notes for fast_shade2.)*
7. Calculate the new palette index:
   $$ v_{new} = v - d $$
8. Clamp $v_{new}$ to the lower bound of its tone group:
   $$ \text{if } v_{new} < \text{color\_margin}, \text{ then } v_{new} = \text{color\_margin} $$
9. Write the modified index back to the grid:
   $$ \text{set\_voxel\_pal}(x, y, z, v_{new}) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_shade2(voxel_grid, x_size, y_size, z_size):
    # Iterate excluding boundaries to avoid index checks for neighbors
    for x in range(1, x_size - 1):
        for y in range(1, y_size - 1):
            for z in range(1, z_size - 1):
                voxel = voxel_grid[x, y, z]
                
                # Skip empty voxels
                if voxel == 0:
                    continue
                
                # Calculate the lower bound of the current 32-index tone group
                color_margin = ((voxel - 1) // 32) * 32 + 1
                
                shading_offset = 0
                
                # Check neighbors and accumulate shading weight
                # Weight 3: Directly above
                if voxel_grid[x, y + 1, z] != 0:
                    shading_offset += 3
                
                # Weight 2: Upper neighbors in X and Z directions
                if voxel_grid[x + 1, y + 1, z] != 0:
                    shading_offset += 2
                if voxel_grid[x - 1, y + 1, z] != 0:
                    shading_offset += 2
                if voxel_grid[x + 1, y + 1, z + 1] != 0:
                    shading_offset += 2
                if voxel_grid[x + 1, y + 1, z - 1] != 0:
                    shading_offset += 2
                
                # Weight 1: Side neighbors in X and Z directions
                if voxel_grid[x + 1, y, z + 1] != 0:
                    shading_offset += 1
                if voxel_grid[x - 1, y, z - 1] != 0:
                    shading_offset += 1
                if voxel_grid[x + 1, y, z - 1] != 0:
                    shading_offset += 1
                if voxel_grid[x - 1, y, z + 1] != 0:
                    shading_offset += 1
                
                # Apply shading
                new_voxel = voxel - shading_offset
                
                # Clamp to the bottom of the current tone group
                if new_voxel < color_margin:
                    new_voxel = color_margin
                
                voxel_grid[x, y, z] = new_voxel
```

## Subtopics

### fast_shade1
A faster but less visually detailed variant of fast_shade2, also assuming vertical sun rays [S1, unofficial]. It uses fewer neighbor checks: it subtracts 3 if $(x, y+1, z)$ is occupied, 2 if $(x+1, y+1, z)$ is occupied, and 1 if $(x+1, y, z+1)$ or $(x-1, y, z-1)$ is occupied [S1, unofficial]. It shares the same tone-group clamping logic and iteration bounds as fast_shade2 [S1, unofficial]. Choose this variant when performance is critical and the reduced shadow detail is acceptable.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the grid, as each voxel is visited once. The constant factor depends on the number of neighbor checks (10 for fast_shade2, 4 for fast_shade1).
- **Memory Complexity:** $O(1)$ auxiliary space, as shading is applied in-place.
- **Performance:** Designed for CPU-only rendering, making it suitable for embedded systems or legacy hardware [S1, unofficial].
- **Strengths:** Extremely fast, simple to implement, no floating-point arithmetic required.
- **Limitations:** Only simulates vertical lighting; no dynamic light sources; no smooth gradients or specular highlights; limited shadow detail compared to ray-marched lighting.
- **When to use:** Real-time voxel engines where lighting is static or pre-computed, and CPU resources are limited.
- **When not to use:** Scenes requiring dynamic lighting, complex materials, or high-fidelity shadows.

## Background
Fast Shading for Voxel Engines was developed as part of the DOOMMY voxel engine, a CPU-based voxel game engine using Lua for scripting [S1, unofficial]. The engine was created by jabberx around 2012 [S1, unofficial]. The method was designed to provide basic lighting effects without the computational cost of ray marching or complex shading models, fitting the constraints of a pure CPU renderer.

## Sources
[S1] jabberx/doommy, https://github.com/jabberx/doommy, fetched 2026-10-27 (unofficial)
