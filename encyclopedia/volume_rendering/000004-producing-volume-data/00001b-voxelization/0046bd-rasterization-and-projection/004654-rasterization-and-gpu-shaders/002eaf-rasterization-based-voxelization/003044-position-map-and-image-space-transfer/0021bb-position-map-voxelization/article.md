# Position Map Voxelization

Also called: Voxelization via Position Map and 3D Image Store

## Overview
Position Map Voxelization is a post-processing step that reads a 2D position map containing world-space positions of surface fragments and writes active voxel indicators into a 3D volume texture [S1, unofficial]. It marks the primary voxel and its 8 immediate neighbors as active to ensure connectivity or fill gaps [S1, unofficial]. It differs from direct rasterization methods by decoupling surface identification from volume population, allowing for gap-filling connectivity via neighbor marking [S1, unofficial]. It builds on [Position Map and Image-Space Transfer](6237-position-map-and-image-space-transfer.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D image texture `positionMap` (binding point 1, format `rgba32f`) storing `vec4` world-space positions, where the alpha channel indicates validity [S1, unofficial].
*   **Output:** A 3D image texture `volume` (binding point 0, format `r8`) storing 8-bit voxel indicators [S1, unofficial].

**Data Structures:**
*   `positionMap`: `layout(binding=1, rgba32f) uniform image2D`. Each texel stores `(x, y, z, alpha)` where `xyz` is the world position and `alpha > 0` indicates an active surface point [S1, unofficial].
*   `volume`: `layout(binding=0, r8) uniform image3D`. Stores voxel activity status [S1, unofficial].

**Parameters:**
*   `voxelDim`: Resolution of the volume, assumed uniform across axes for the conversion formula [S1, unofficial].
*   `xBounds`, `yBounds`, `zBounds`: Volume bounds defined as `(min, max)` pairs for each axis [S1, unofficial].
*   `stepSize`: Distance in world units to offset positions when calculating neighbor voxels [S1, unofficial].

**Steps:**
1. Bind the voxelizer shader, the `positionMap` as a read-only image texture at binding point 1, and the `volume` as a read-write image texture at binding point 0 [S1, unofficial].
2. Bind a full-screen quad VAO and load identity matrices for projection, view, and model matrices [S1, unofficial].
3. Disable depth testing, depth mask, culling, and color mask to prevent incorrect overwrites during the update [S1, unofficial].
4. Draw the full-screen quad, triggering the fragment shader for each pixel [S1, unofficial].
5. Generate mipmaps for the 3D volume texture [S1, unofficial].
6. Unbind textures, shader, and VAO, and reset OpenGL state (enable depth test, culling, depth mask, color mask) [S1, unofficial].

**Fragment Shader Logic:**
For each fragment corresponding to a texel in `positionMap`:
1. Load the world-space position: `worldPos = imageLoad(positionMap, ivec2(gl_FragCoord.xy))` [S1, unofficial].
2. Check if the voxel is active by testing `worldPos.a > 0` [S1, unofficial].
3. If active, calculate the voxel index using the conversion formula and store `vec4(1)` (truncated to `1.0` in `r8`) in the volume texture at that index [S1, unofficial].
4. For each of the 8 neighbors defined by sign combinations of $(\pm 1, \pm 1, \pm 1)$:
    * Calculate the neighbor's world position: `neighborPos = worldPos.xyz + stepSize * normalize(vec3(dx, dy, dz))` where `dx, dy, dz \in \{-1, 1\}` [S1, unofficial].
    * Calculate the neighbor's voxel index using the conversion formula [S1, unofficial].
    * Store `vec4(1)` in the volume texture at the neighbor's index [S1, unofficial].

**Index Conversion Formula:**
Given volume bounds $[x_{min}, x_{max}]$, $[y_{min}, y_{max}]$, $[z_{min}, z_{max}]$ and resolution `voxelDim`, the voxel indices $(x, y, z)$ are calculated as:

$$
x = \text{voxelDim} \times \frac{pos.x - x_{min}}{x_{max} - x_{min}}
$$

$$
y = \text{voxelDim} \times \frac{pos.y - y_{min}}{y_{max} - y_{min}}
$$

$$
z = \text{voxelDim} \times \frac{pos.z - z_{min}}{z_{max} - z_{min}}
$$

The function returns `ivec3(x, y, z)` [S1, unofficial]. Standard integer truncation or floor is implied by the return type `ivec3` in typical GLSL implementations [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def position_map_voxelization(position_map, volume, voxel_dim, bounds, step_size):
    # Disable depth testing, depth mask, culling, and color mask
    state.depth_test = False
    state.depth_mask = False
    state.culling = False
    state.color_mask = False
    
    # Bind voxelizer shader, position_map as read-only at binding 1, volume as read-write at binding 0
    bind_shader(voxelizer_shader)
    bind_image_texture(position_map, binding=1, access=READ_ONLY)
    bind_image_texture(volume, binding=0, access=READ_WRITE)
    
    # Bind full-screen quad VAO and load identity matrices
    bind_full_screen_quad()
    load_identity_matrices()
    
    # Draw full-screen quad to trigger fragment shader per pixel
    draw_full_screen_quad()
    
    # Generate mipmaps for the 3D volume texture
    generate_mipmaps(volume)
    
    # Unbind textures, shader, and VAO
    unbind_all()
    
    # Reset OpenGL state: enable depth test, culling, depth mask, color mask
    state.depth_test = True
    state.culling = True
    state.depth_mask = True
    state.color_mask = True

def voxelizer_fragment_shader(position_map, volume, voxel_dim, bounds, step_size):
    # 1. Read world-space position from position map
    frag_coord = get_frag_coord()
    world_pos_data = image_load(position_map, ivec2(frag_coord))
    world_pos = world_pos_data.xyz
    alpha = world_pos_data.w

    # 2. Check if active
    if alpha > 0:
        # 3. Convert to voxel index and store
        voxel_idx = calculate_voxel_index(world_pos, voxel_dim, bounds)
        image_store(volume, voxel_idx, vec4(1.0))

        # 4. Mark 8 neighbors
        for dx in [-1, 1]:
            for dy in [-1, 1]:
                for dz in [-1, 1]:
                    # Calculate neighbor world position
                    direction = normalize(vec3(dx, dy, dz))
                    neighbor_pos = world_pos + step_size * direction
                    
                    # Convert to voxel index and store
                    neighbor_idx = calculate_voxel_index(neighbor_pos, voxel_dim, bounds)
                    image_store(volume, neighbor_idx, vec4(1.0))

def calculate_voxel_index(pos, voxel_dim, bounds):
    # Convert world position to voxel index using bounds and resolution
    x = voxel_dim * (pos.x - bounds['x'][0]) / (bounds['x'][1] - bounds['x'][0])
    y = voxel_dim * (pos.y - bounds['y'][0]) / (bounds['y'][1] - bounds['y'][0])
    z = voxel_dim * (pos.z - bounds['z'][0]) / (bounds['z'][1] - bounds['z'][0])
    
    # Truncate to integer indices
    return ivec3(int(x), int(y), int(z))
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Positions outside the volume bounds result in negative or out-of-bounds indices, requiring explicit clamping [G].
*   **Precision:** Insufficient precision in the position map leads to incorrect voxel indexing [G].
*   **Aliasing:** Solid voxelization of closed polygonal models may exhibit aliasing effects, appearing as prolonged lines where some voxels are missing at the rasterization stage [G].
*   **Step Size Tuning:** The `stepSize` parameter must be carefully chosen; too small a value may leave gaps between neighbors, while too large a value may cause excessive overlap or false voxels [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Performance:** Not given in the sources [S1, unofficial].
*   **Strengths:** Ensures connectivity by marking neighbors; decouples surface rasterization from volume population [S1, unofficial].
*   **Limitations:** Requires a pre-generated position map; neighbor marking may over-approximate the surface, filling gaps but potentially adding false voxels [G].
*   **When to use:** Not given in the sources [S1, unofficial].
*   **When not to use:** Not given in the sources [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
