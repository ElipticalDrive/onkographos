# Sparse Brickmap Volume Rendering

Also called: Micro Voxel Volume Rendering, Two-Level DDA Raymarching

## Overview
Sparse Brickmap Volume Rendering is a voxel rendering technique that uses a two-level data structure (coarse brick grid and fine voxel bricks) to render sparse volumes efficiently, integrating hardware depth for correct compositing with rasterized geometry [S1, unofficial]. It builds on [Sparse Page Table and Brick Pool](2550-sparse-page-table-and-brick-pool.md) by replacing the generic page-table indirection with a fixed $8^3$ brick hierarchy optimized for two-level Digital Differential Analyzer (DDA) traversal [S1, unofficial]. Unlike dense voxel grids, it significantly reduces memory footprint by collapsing fully-buried terrain into uniform entries and storing only mixed (surface) bricks in a compact pool [S1, unofficial].

## How it works
**Inputs:**
- Camera position and ray direction (from screen coordinates).
- Coarse Index Grid: Array of `uint32`, one entry per $8 \times 8 \times 8$ brick cell.
- Brick Pool: Compact array of stored voxel data for 'mixed' bricks only.
- Voxel Size: Default 5cm [S1, unofficial].
- Grid Size: Configurable runtime dimensions of $128^3$, $256^3$, or $512^3$ [S1, unofficial].
- Parameters: `aoStrength` (0 disables AO), sun direction, ambient intensities.

**Outputs:**
- Fragment color (albedo, lighting, AO).
- Hardware depth value (distance to hit point).

**Data Structures:**
1.  **Coarse Index Grid**: A 3D array of `uint32`. Each entry corresponds to an $8 \times 8 \times 8$ brick cell in the logical volume.
    -   Value `0xFFFFFFFF`: Empty sentinel. The brick is empty.
    -   Value `0x80000000 | material_id`: Uniform-material sentinel. The brick is fully solid with a single material.
    -   Value `index < 0x80000000`: Index into the Brick Pool. The brick contains mixed data.
2.  **Brick Pool**: A 1D array of voxel data blocks. Each block stores the $8 \times 8 \times 8$ voxels for a mixed brick. Fully-buried or empty bricks do not occupy pool memory [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Initialization**: For each pixel, compute the ray origin and direction. Initialize `t` (distance along ray) to 0.

2.  **Coarse DDA Traversal**:
    -   Step through the Coarse Index Grid along the ray using DDA.
    -   For each coarse cell $(i, j, k)$ encountered:
        -   Let `entry` be the value at `CoarseIndexGrid[i][j][k]`.
        -   If `entry == 0xFFFFFFFF` (Empty): Skip the entire $8 \times 8 \times 8$ brick cell in one step. Continue to the next coarse cell [S1, unofficial].
        -   If `entry == 0x80000000 | material_id` (Uniform):
            -   The ray hits the entry face of this brick immediately.
            -   No fine data fetch is performed.
            -   Set hit material to `material_id`.
            -   Calculate hit position and normal at the brick entry face.
            -   Proceed to **Lighting & Shading** (Step 5).
        -   If `entry < 0x80000000` (Mixed):
            -   Fetch the brick data from `BrickPool[entry]`.
            -   Perform **Fine DDA Traversal** (Step 3).

3.  **Fine DDA Traversal**:
    -   Using the fetched brick data, perform a fine-grained DDA traversal through the $8 \times 8 \times 8$ voxels within that brick.
    -   Find the first non-empty voxel hit along the ray segment within the brick.
    -   If a hit is found:
        -   Calculate the precise hit position and normal.
        -   Set hit material based on the voxel type.
        -   Proceed to **Lighting & Shading** (Step 5).
    -   If no hit is found within the brick, continue Coarse DDA to the next cell.

4.  **Depth Output**:
    -   Write the distance to the hit point (`t`) to the hardware depth buffer. This ensures voxels depth-test correctly against rasterized geometry and subsequent passes (sky, fog, bloom) [S1, unofficial].

5.  **Lighting & Shading**:
    -   **Sun Shadow Raycast**: From the hit point, cast a ray towards the sun direction. If the ray intersects any opaque voxel before reaching the sun, the point is in shadow [S1, unofficial].
    -   **Hemisphere Ambient Lighting**: Calculate ambient lighting based on the surface normal $\mathbf{n}$.
        -   If $\mathbf{n} \cdot \mathbf{up} > 0$ (pointing up), use sky-tinted color.
        -   If $\mathbf{n} \cdot \mathbf{up} \le 0$ (pointing down), use ground-bounce color.
    -   **Per-Voxel Albedo Variation**: Apply hash-based noise to the voxel's base albedo to add visual detail and reduce uniformity [S1, unofficial].
    -   **Corner Ambient Occlusion (AO)**:
        -   At the primary hit, sample the 8 neighboring voxels around the hit face.
        -   Calculate the solidity of these neighbors.
        -   Apply bilinear interpolation across the face for smooth per-pixel AO.
        -   Attenuate hemisphere ambient fully.
        -   Darken direct lighting by 30% (if not in shadow) or apply a stylized darkening factor controlled by `aoStrength` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxel_pixel(ray_origin, ray_direction, coarse_grid, brick_pool, voxel_size, ao_strength, sun_direction):
    t = 0.0
    hit_material = None
    hit_position = None
    hit_normal = None
    
    # Coarse DDA
    coarse_step = compute_coarse_dda_step(ray_origin, ray_direction, voxel_size * 8.0)
    coarse_pos = ray_origin
    
    while t < max_distance:
        # Get current coarse cell index
        i, j, k = get_coarse_cell_index(coarse_pos, voxel_size * 8.0)
        entry = coarse_grid[i][j][k]
        
        if entry == 0xFFFFFFFF:
            # Empty sentinel: skip entire brick
            coarse_pos += coarse_step
            t += length(coarse_step)
            continue
            
        if (entry & 0x80000000) != 0:
            # Uniform-material sentinel
            material_id = entry & 0x7FFFFFFF
            # Hit entry face of the brick
            hit_position = coarse_pos
            hit_normal = compute_face_normal(coarse_pos, ray_direction)
            hit_material = material_id
            break
            
        # Mixed brick: fetch from pool
        brick_data = brick_pool[entry]
        
        # Fine DDA within brick
        local_ray_origin = ray_origin - get_brick_origin(i, j, k)
        local_ray_direction = ray_direction
        brick_size = voxel_size * 8.0
        
        fine_hit_t, fine_hit_pos, fine_hit_normal, fine_material = 
            traverse_fine_brick(local_ray_origin, local_ray_direction, brick_data, voxel_size)
            
        if fine_hit_t is not None:
            # Transform back to world space
            hit_position = get_brick_origin(i, j, k) + fine_hit_pos
            hit_normal = fine_hit_normal
            hit_material = fine_material
            t = length(hit_position - ray_origin)
            break
        else:
            # No hit in this brick, continue coarse traversal
            coarse_pos += coarse_step
            t += length(coarse_step)
            
    if hit_material is None:
        return background_color, 0.0
        
    # Lighting
    shadow = cast_shadow_ray(hit_position, sun_direction, coarse_grid, brick_pool, voxel_size)
    ambient = compute_hemisphere_ambient(hit_normal)
    albedo = apply_albedo_variation(hit_material, hit_position)
    
    # Corner AO
    ao_factor = compute_corner_ao(hit_position, hit_normal, coarse_grid, brick_pool, voxel_size, ao_strength)
    ambient *= ao_factor
    direct_light = compute_direct_light(hit_material, hit_normal, sun_direction, shadow) * ao_factor * 0.7 # 30% darkening
    
    color = ambient + direct_light
    depth = t
    
    return color, depth

def traverse_fine_brick(ray_origin, ray_direction, brick_data, voxel_size):
    # Standard DDA through 8x8x8 voxels
    # Returns (t, position, normal, material) or (None, None, None, None)
    # Implementation details omitted for brevity, follows standard voxel DDA
    pass

def compute_corner_ao(hit_position, hit_normal, coarse_grid, brick_pool, voxel_size, ao_strength):
    if ao_strength == 0:
        return 1.0
        
    # Sample 8 neighbors around hit face
    neighbors = get_8_neighbors(hit_position, voxel_size)
    solidity = compute_solidity(neighbors, coarse_grid, brick_pool, voxel_size)
    
    # Bilinear interpolation across face for smooth AO
    ao_factor = interpolate_ao(solidity, hit_position)
    
    # Attenuate ambient fully, darken direct by 30%
    # The 30% darkening is applied in the main shading step
    return ao_factor
```

## Subtopics

### [Sparse Volume Rendering with Block Walking and Culling](3041-sparse-volume-rendering-with-block-walking-and-culling.md)
A method to render sparse volumetric data using hardware ray tracing that accelerates traversal by storing adjacency information in BVH nodes to skip ray-tracing queries for adjacent blocks (Block Walking) and using density-based culling masks to skip irrelevant blocks (Block Culling). It builds on [Sparse Brickmap Volume Rendering](2978-sparse-brickmap-volume-rendering.md) by replacing the brick-based hierarchy with a two-level block/subblock structure optimized for hardware ray query acceleration.

## Operating characteristics
- **Time Complexity**: Coarse DDA skips empty regions in $O(1)$ per brick. Fine DDA is $O(V)$ per mixed brick, where $V=8^3$. Overall complexity depends on the density of mixed bricks.
- **Memory Complexity**: Significantly reduced compared to dense voxel grids. Only mixed bricks occupy the Brick Pool. Fully-buried terrain collapses to uniform entries in the Coarse Index Grid [S1, unofficial].
- **Measured Performance**: Validated against a brute-force fixed-step reference raymarcher over 15k random rays. The two-level DDA with pool indirection and uniform-brick early hits produces identical results to the previous dense traversal, with only 0.02% disagreement on grazing-corner cases [S1, unofficial].
- **Strengths**: Efficient memory usage for sparse volumes; correct hardware depth integration; fast traversal via coarse skipping; visual quality enhanced by AO and albedo variation.
- **Limitations**: Fixed $8 \times 8 \times 8$ brick size may not be optimal for all scales; grazing-corner cases have minor inaccuracies (0.02% disagreement) [S1, unofficial].
- **When to use**: When rendering sparse volumetric data (e.g., terrain, caves) with hardware depth integration and efficient memory usage is required.
- **When not to use**: When uniform brick size is inappropriate or when exact pixel-perfect consistency with fixed-step raymarching is critical (due to grazing-corner edge cases).

## Background
Sparse Brickmap Volume Rendering was implemented in the 'project-vapor' engine using Metal-cpp and Vulkan, specifically in the file `3d_voxel_raymarch.metal` [S1, unofficial]. It was introduced as an experimental fullscreen voxel raymarching pass that renders a sparse brickmap volume between the main geometry pass and sky rendering [S1, unofficial]. The technique was developed to replace dense voxel grids with a sparse brickmap structure, significantly reducing memory footprint and improving performance [S1, unofficial]. The implementation includes procedural demo volume generation using Perlin noise with FBM, cave carving, and ore speckles [S1, unofficial].

## Sources
[S1] Add experimental micro voxel volume rendering with sparse brickmap, https://github.com/painfulexistence/project-vapor/pull/61, fetched 2026-10-25 (unofficial)
