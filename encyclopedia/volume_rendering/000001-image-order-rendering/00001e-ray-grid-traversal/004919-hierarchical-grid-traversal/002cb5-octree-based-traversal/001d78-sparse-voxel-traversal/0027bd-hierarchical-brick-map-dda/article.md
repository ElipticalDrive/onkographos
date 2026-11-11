# Hierarchical Brick-Map DDA

Also called: Multi-level DDA with brick map, Sparse brick-map ray marching

## Overview
Hierarchical Brick-Map DDA is a ray traversal technique that replaces dense voxel buffers with a sparse, hierarchical structure to skip empty regions in volumetric data [S1, unofficial]. It operates by maintaining a top-level grid of cells (typically stride 8 voxels) that point to allocated "bricks" containing fine-grained voxel data; if a top-level cell is empty, the ray strides over the entire cell volume, descending only into occupied bricks for local DDA [S1, unofficial]. This approach significantly reduces memory footprint and traversal time for sparse volumes, such as drone airspace which is >99% empty, compared to dense voxel storage or non-hierarchical sparse methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A dense voxel grid (or SDF field) defined on a global coordinate system; ray origin $O$ and direction $D$.
*   **Output:** Ray-marched color/depth, with traversal skipping empty space.

**Data Structures:**
The method relies on three primary GPU buffers and a uniform structure [S1, unofficial]:
1.  **`topGrid`**: A sparse 3D grid where each cell contains a pointer (index) to a brick in the `brickPool`, or a null value if the cell is empty. The dimensions are defined by `top_size`.
2.  **`brickPool`**: A contiguous array of occupied bricks. Each brick stores $BRICK^3$ voxel values.
3.  **`counter`**: An atomic counter used during the build phase to allocate slots in `brickPool`.
4.  **`Grid` Uniform**: Contains `top_size` (dimensions of the top-level grid) and `BRICK` (side length of a brick in voxels, typically 8) [S1, unofficial].

**Parameters:**
*   $BRICK$: Side length of a voxel brick. Default value is 8 [S1, unofficial].
*   $MAX\_STEPS$: Maximum number of ray marching steps. Increased to 1024 to accommodate hierarchical overhead and longer paths in sparse data (up from 512 in dense implementations) [S1, unofficial].

**Build Phase (Compute Shader):**
Before traversal, the sparse structure is built from dense data [S1, unofficial]:
1.  For each top-level cell in the global grid:
2.  Scan the $BRICK^3$ voxels corresponding to that cell.
3.  If any voxel is non-zero (solid), atomically allocate a slot in `brickPool` using `counter`.
4.  Copy the voxel data into the allocated slot.
5.  Update the corresponding entry in `topGrid` to point to the new brick index.
6.  *Optimization:* Early-exit the scan loop upon finding the first solid voxel to reduce unnecessary reads [S1, unofficial].

**Traversal Phase (Ray Marching Shader):**
The traversal uses a multi-level Digital Differential Analyzer (DDA) [S1, unofficial]:

1.  **Initialization:** Compute initial ray parameters ($t_{min}, t_{max}$) and step sizes. Initialize top-level cell coordinates $(x, y, z)$ based on ray origin.
2.  **Top-Level Traversal Loop:**
    *   While $t < t_{max}$ and steps $< MAX\_STEPS$:
    *   Check if the current top-level cell $(x, y, z)$ is occupied (pointer in `topGrid` is not null).
    *   **If Empty:**
        *   Calculate the distance to the next top-level cell boundary along the ray direction. Let this be $\Delta t_{top}$.
        *   Update $t \leftarrow t + \Delta t_{top}$.
        *   Update $(x, y, z)$ to the next top-level cell indices.
        *   Continue loop.
    *   **If Occupied:**
        *   **Descent:** Retrieve the brick index from `topGrid`. Map the ray origin and direction into the local coordinate system of the brick. Re-derive the fine-grained ray direction vector $fv$ based on the actual ray position within the occupied brick to ensure correct sampling [S1, unofficial].
        *   **Fine-Grained DDA:** Perform standard voxel DDA within the brick's local bounds.
        *   **Ascent:** When the ray exits the brick (either hitting a surface or leaving the brick boundary):
            *   Map the exit point back to global coordinates.
            *   Recompute $c_{tmax}$ (distance to the next top-level cell boundary) for the new top cell to resume coarse traversal correctly [S1, unofficial].
            *   Update $(x, y, z)$ to the new top-level cell.
            *   Continue loop.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BRICK = 8
MAX_STEPS = 1024

# Data Structures (conceptual)
# topGrid: array of int, indexed by top-level cell coordinates
# brickPool: array of float, size BRICK^3 per brick
# gridUniform.top_size: vec3i

def ray_march_brick_map(ray_origin, ray_direction, topGrid, brickPool, top_size):
    t = 0.0
    steps = 0
    
    # Initial top-level cell coordinates
    # Assuming integer division/flooring for grid alignment
    tx = floor(ray_origin.x / BRICK)
    ty = floor(ray_origin.y / BRICK)
    tz = floor(ray_origin.z / BRICK)
    
    # Precompute inverse direction for DDA
    inv_dir = 1.0 / ray_direction
    
    # Determine initial step sizes for top-level grid
    # Distance to next integer boundary in global space
    # Note: Simplified logic; real implementation uses tmaxx, tmaxy, tmaxz
    t_max_top_x = (tx + 1.0) * BRICK - ray_origin.x if inv_dir.x > 0 else tx * BRICK - ray_origin.x
    # ... similar for y, z ...
    
    while steps < MAX_STEPS and t < t_max_global:
        steps += 1
        
        # Check if current top-level cell is occupied
        brick_idx = topGrid[tx, ty, tz]
        
        if brick_idx == NULL:
            # Empty cell: stride over the entire brick volume
            # Calculate distance to next top-level boundary
            # This requires careful handling of ray direction signs
            dt = min_distance_to_top_boundary(ray_origin + ray_direction * t, ray_direction, tx, ty, tz, BRICK)
            t += dt
            
            # Advance top-level coordinates
            if ray_direction.x > 0: tx += 1
            elif ray_direction.x < 0: tx -= 1
            # ... similar for y, z ...
            
        else:
            # Occupied cell: Descend into brick
            # 1. Determine local ray origin within the brick
            # Global origin of this brick
            brick_origin_global = vec3(tx * BRICK, ty * BRICK, tz * BRICK)
            
            # Local ray origin relative to brick
            local_ray_origin = (ray_origin + ray_direction * t) - brick_origin_global
            
            # 2. Re-derive fine-grained ray direction fv
            # In many implementations, fv is simply ray_direction, 
            # but the source notes it is "re-derived based on actual ray position"
            # to ensure correct sampling. This often implies handling 
            # floating point precision or specific coordinate space conversions.
            fv = ray_direction 
            
            # 3. Perform Fine-Grained DDA within brick
            # Local brick bounds are [0, BRICK]
            local_t_hit = dda_fine(local_ray_origin, fv, brickPool, BRICK)
            
            if local_t_hit is HIT:
                return local_t_hit
            
            # 4. Ascent: Ray exited brick
            # Calculate distance traveled in local space
            local_dt = length(local_ray_origin + fv * local_t_hit - local_ray_origin) # Simplified
            # Actually, local_t_hit is the t parameter in local space.
            # Global t update:
            t += local_t_hit # Assuming fv is normalized or scaled correctly
            
            # 5. Recompute c_tmax for new top cell
            # The ray is now at global position: ray_origin + ray_direction * t
            # We need to find the next top-level boundary from this new position
            # This effectively resets the top-level DDA state for the new cell
            # The source notes: "c_tmax is recomputed for the new top cell"
            
            # Advance to next top-level cell
            # Determine which face of the current brick was exited
            # and increment/decrement tx, ty, tz accordingly
            # This logic is complex and depends on the specific DDA implementation
            # For simplicity, we assume the ray exits and we re-evaluate top-level step
            # In a robust implementation, this is handled by the top-level DDA state
            # being reset or updated based on the exit face.
            
            # Re-calculate top-level step size from current global position
            # This is the "c_tmax" recomputation
            # ... (Implementation detail omitted as source does not provide exact formula) ...
            
            # For the sake of the loop structure, we assume tx, ty, tz are updated
            # to the cell the ray is now entering or adjacent to.
            # If the ray exited the brick but is still within the same top-level cell 
            # (unlikely if brick == top cell), we stay. 
            # If brick < top cell, we might need to re-enter the same top cell.
            # However, the source implies "ascends back to the top-level grid",
            # suggesting the top-level DDA continues.
            
            # To keep pseudocode valid and self-contained, we assume the 
            # top-level coordinates are updated to the next cell the ray will traverse.
            # This is typically done by checking the exit face of the brick 
            # and incrementing the corresponding top-level coordinate.
            
            # Example: if ray exited +X face of brick
            if exit_face == +X:
                tx += 1
            elif exit_face == -X:
                tx -= 1
            # ... similar for y, z ...
            
            # Note: The source mentions "c_tmax is recomputed". 
            # This implies the top-level DDA state (tmaxx, tmaxy, tmaxz) 
            # must be updated based on the new global position.
            # This is a standard DDA update step.
            
    return NO_HIT
```

**Edge Cases and Pitfalls:**
*   **Brick Edge Transitions:** Visual popping or missing voxels can occur at brick boundaries if the transition between top-level cells and brick interiors is not handled smoothly. Validation requires toggling camera angles to ensure clean boundary-crossing transitions [S1, unofficial].
*   **Axis-Aligned Rays:** The method reuses AABB (Axis-Aligned Bounding Box) clipping and safe-ray-distance primitives from dense implementations to handle axis-aligned rays and rays outside the grid volume correctly without artifacts [S1, unofficial].

## Subtopics

### [Sparse Ray Marching with Nested Axis-Aligned Distance Fields (NAADF)](6260-sparse-ray-marching-with-nested-axis-aligned-distance-fields.md)
Sparse Ray Marching with Nested Axis-Aligned Distance Fields (NAADF) is a technique for traversing sparse voxel grids by skipping over empty space using precomputed distance fields, replacing traditional brick coordinate rasterization with a simpler pipeline suitable for g-buffer and path tracing. It differs from its sibling [Hierarchical Brick-Map DDA](4105-hierarchical-brick-map-dda.md) by calculating the largest empty region along the ray's path via axis-aligned bounds, allowing it to step over multiple cells at once rather than advancing one cell boundary per iteration.

## Operating characteristics
*   **Time Complexity:** $O(k)$ where $k$ is the number of occupied bricks intersected by the ray, plus the cost of traversing empty top-level cells. In sparse scenes, $k \ll N^3$ (where $N$ is the global resolution).
*   **Memory Complexity:** $O(B \cdot BRICK^3)$ where $B$ is the number of occupied bricks. For a heightmap stand-in with $N=128$, the brick count is roughly $top^2$ (~256 bricks) instead of the worst-case $top^3$ (4096) [S1, unofficial]. Memory savings grow with sparsity.
*   **Measured Performance:** For drone airspace (>99% empty), the method significantly reduces traversal time compared to dense voxel buffers [S1, unofficial].
*   **Strengths:** Significant memory reduction in sparse volumes; efficient skipping of empty space; compatible with GPU compute shaders for build phase.
*   **Limitations:** Requires a build phase to construct the sparse structure; potential for visual artifacts at brick boundaries if not implemented carefully; increased $MAX\_STEPS$ may be required due to hierarchical overhead.
*   **When to Use:** Sparse volumetric data (e.g., drone airspace, large outdoor scenes with sparse objects).
*   **When Not to Use:** Dense volumetric data (e.g., medical CT scans with high fill rate) where the overhead of hierarchical traversal may not justify the memory savings, or when dynamic updates are frequent and costly.

## Background
Hierarchical Brick-Map DDA was introduced as part of the ResQ Viz project, specifically in PR #67, to optimize ray marching for sparse volumes like drone airspace [S1, unofficial]. It builds upon the concept of sparse voxel traversal and brick-based storage, adapting it for GPU-based ray marching with WebGPU. The method addresses the inefficiency of dense voxel buffers in highly sparse environments by introducing a two-level hierarchy: a coarse top-level grid and fine-grained bricks.

## Sources
[S1] feat(viz): hierarchical brick-map DDA for /spike.html, https://github.com/resq-software/viz/pull/67, fetched 2026-10-25 (unofficial)
[S2] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
