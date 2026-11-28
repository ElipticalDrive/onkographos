# MultiDDA

Also called: 2-Level DDA (Brick Grid), BrickMap traversal, Coarse-to-Fine Traversal, Nested DDA for hierarchical grids, Outer/Inner DDA, Two-Level Sparse Voxel Grid Traversal, Two-level DDA with brickmap

## Overview
MultiDDA is a ray traversal algorithm for sparse voxel grids that uses nested Digital Differential Analyzers (DDA) to skip empty space efficiently [S2, S3, unofficial]. It solves the performance bottleneck of naive DDA in large, sparse scenes by first traversing a coarse "brick" grid to identify occupied regions, and only then performing a fine-grained voxel-level traversal within those regions [S3, unofficial]. It differs from its parent [Voxel Ray Marching with Cube-Level DDA](3069-voxel-ray-marching-with-cube-level-dda.md) by explicitly decoupling the traversal into two distinct DDA loops (brick-level and voxel-level) rather than a single unified state machine, and from [Efficient Voxel Marking for Hierarchical Volumetric Fusion](12765-efficient-voxel-marking-for-hierarchical-volumetric-fusion.md) by focusing on rendering acceleration rather than volumetric fusion [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $O$, ray direction $D$, inverse direction $D^{-1}$, brick grid dimensions $N_b \times N_b \times N_b$, brick size $B$ (typically $8 \times 8 \times 8$ voxels), voxel grid texture $T_v$, brick occupancy texture $T_b$.
- Output: The index of the first occupied voxel along the ray, or 0 if the ray exits the grid without hitting occupied space.

**Data Structures:**
1.  **Brick Grid ($T_b$):** A 3D texture (`Texture3D<uint>`) where each texel represents an $B \times B \times B$ block of voxels. A value $>0$ indicates the brick contains at least one occupied voxel; $0$ indicates it is empty [S3, unofficial].
2.  **Voxel Grid ($T_v$):** A 3D texture storing the actual voxel data (e.g., color, density) [S3, unofficial].

**Parameters:**
-   $B$: Brick side length in voxels. Typical value: 8 [S3, unofficial].
-   $N_b$: Side length of the brick grid in bricks.

**Algorithm Steps:**

1.  **Initialize Outer DDA:**
    Calculate the step direction and delta for the brick grid. Let $D^{-1} = (1/D_x, 1/D_y, 1/D_z)$.
    Determine the initial brick cell $(c_x, c_y, c_z)$ containing the ray origin.
    Calculate $t_{next}$, the distance to the next brick boundary in each axis:
    $$ t_{next} = \begin{cases} (c_x + 1 - O_x/D_x) \cdot D_x & \text{if } D_x > 0 \\ (c_x - O_x/D_x) \cdot D_x & \text{if } D_x < 0 \end{cases} $$
    (Analogous for $y$ and $z$).
    Calculate $t_{delta}$, the distance between brick boundaries:
    $$ t_{delta} = B \cdot |D| \quad \text{(component-wise: } t_{delta, x} = B \cdot |D_x| \text{)} $$
    Determine step direction $S = \text{sign}(D)$ [S3, unofficial].

2.  **Traverse Brick Grid:**
    Loop for a maximum number of iterations (e.g., $3 \cdot N_b$ to handle diagonal traversal across the grid) [S3, unofficial]:
    a.  **Boundary Check:** If any component of the current brick cell $(c_x, c_y, c_z)$ is outside $[0, N_b-1]$, return 0 (ray exited grid) [S3, unofficial].
    b.  **Occupancy Test:** Load the brick occupancy value $v_b = T_b.Load(c_x, c_y, c_z)$ [S3, unofficial].
    c.  **Inner Traversal:** If $v_b > 0$:
        i.  Call `trace_brick` with the ray parameters and current brick cell.
        ii. If `trace_brick` returns a valid voxel index $v > 0$, return $v$ [S3, unofficial].
    d.  **Advance Outer DDA:**
        Find the next axis to cross: $axis = \text{argmin}(t_{next})$ (excluding the current axis if it's the minimum, effectively picking the smallest positive $t_{next}$) [S3, unofficial].
        Update $t_{next}$ for the crossed axis: $t_{next, axis} += t_{delta, axis}$ [S3, unofficial].
        Update brick cell: $c_{axis} += S_{axis}$ [S3, unofficial].

3.  **Inner Traversal (`trace_brick`):**
    Given ray $O, D, D^{-1}$ and brick cell $C_b$:
    a.  **Local Ray Setup:** Transform the ray into the local coordinate system of the brick.
        $O_{local} = O - C_b \cdot B$
        $D_{local} = D$
        $D^{-1}_{local} = D^{-1}$
    b.  **Initialize Voxel DDA:**
        Calculate initial voxel cell $(v_x, v_y, v_z)$ within the brick containing $O_{local}$.
        Calculate $t_{next, v}$ and $t_{delta, v}$ for voxel boundaries (where $t_{delta, v} = 1 \cdot |D|$) [S3, unofficial].
    c.  **Voxel Loop:**
        Loop while $t_{next, v} < t_{exit\_brick}$ (distance to the far side of the brick):
        i.  Check if voxel $(v_x, v_y, v_z)$ is occupied in $T_v$.
        ii. If occupied, return the global voxel index.
        iii. Advance voxel DDA:
            $axis = \text{argmin}(t_{next, v})$
            $t_{next, v, axis} += t_{delta, v, axis}$
            $v_{axis} += \text{sign}(D_{axis})$
    d.  If no occupied voxel is found within the brick, return 0 [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_dda_traverse(ray_origin, ray_dir, brick_grid_ext, brick_size, voxel_texture, brick_texture):
    # 1. Initialize Outer DDA (Brick Level)
    inv_dir = 1.0 / ray_dir
    step_dir = int3(sign(ray_dir))
    
    # Initial brick cell
    brick_cell = int3(floor(ray_origin / brick_size))
    
    # Distance to next brick boundary
    t_next = float3(
        (brick_cell.x + 0.5 - ray_origin.x / ray_dir.x) * ray_dir.x if ray_dir.x > 0 else (brick_cell.x - ray_origin.x / ray_dir.x) * ray_dir.x,
        (brick_cell.y + 0.5 - ray_origin.y / ray_dir.y) * ray_dir.y if ray_dir.y > 0 else (brick_cell.y - ray_origin.y / ray_dir.y) * ray_dir.y,
        (brick_cell.z + 0.5 - ray_origin.z / ray_dir.z) * ray_dir.z if ray_dir.z > 0 else (brick_cell.z - ray_origin.z / ray_dir.z) * ray_dir.z
    )
    
    # Delta to next brick boundary
    t_delta_brick = float3(brick_size * abs(ray_dir.x), brick_size * abs(ray_dir.y), brick_size * abs(ray_dir.z))
    
    # Max iterations to prevent infinite loops in case of logic errors
    max_iter = 3 * brick_grid_ext
    
    for i in range(max_iter):
        # Boundary check
        if any(brick_cell < 0) or any(brick_cell >= brick_grid_ext):
            return 0
            
        # Check brick occupancy
        brick_occupancy = brick_texture.load(brick_cell)
        
        if brick_occupancy > 0:
            # Inner traversal
            voxel_idx = trace_brick(ray_origin, ray_dir, brick_cell, brick_size, voxel_texture)
            if voxel_idx > 0:
                return voxel_idx
        
        # Advance Outer DDA
        # Find the axis with the smallest t_next
        min_t = min(t_next.x, min(t_next.y, t_next.z))
        axis_mask = step(t_next, min_t) # Returns 1.0 where t_next == min_t, 0.0 otherwise
        
        # Update t_next for the crossed axis
        t_next += axis_mask * t_delta_brick
        
        # Update brick cell
        brick_cell += int3(axis_mask) * step_dir
        
    return 0

def trace_brick(ray_origin, ray_dir, brick_cell, brick_size, voxel_texture):
    # Transform ray to local brick space
    origin_local = ray_origin - brick_cell * brick_size
    dir_local = ray_dir
    inv_dir_local = 1.0 / dir_local
    
    # Initial voxel cell within the brick
    voxel_cell = int3(floor(origin_local / 1.0)) # Voxel size is 1.0
    
    # Clamp to brick bounds [0, brick_size-1]
    voxel_cell = clamp(voxel_cell, 0, brick_size - 1)
    
    # Distance to next voxel boundary
    t_next_v = float3(
        (voxel_cell.x + 1.0 - origin_local.x / dir_local.x) * dir_local.x if dir_local.x > 0 else (voxel_cell.x - origin_local.x / dir_local.x) * dir_local.x,
        (voxel_cell.y + 1.0 - origin_local.y / dir_local.y) * dir_local.y if dir_local.y > 0 else (voxel_cell.y - origin_local.y / dir_local.y) * dir_local.y,
        (voxel_cell.z + 1.0 - origin_local.z / dir_local.z) * dir_local.z if dir_local.z > 0 else (voxel_cell.z - origin_local.z / dir_local.z) * dir_local.z
    )
    
    t_delta_voxel = float3(abs(dir_local.x), abs(dir_local.y), abs(dir_local.z))
    
    # Max distance within the brick (approximate, using brick diagonal or max extent)
    # A safer bound is the distance to the far side of the brick along the ray
    # For simplicity, we limit iterations to brick_size * 3
    max_voxel_iter = 3 * brick_size
    
    for i in range(max_voxel_iter):
        # Boundary check within brick
        if any(voxel_cell < 0) or any(voxel_cell >= brick_size):
            return 0
            
        # Check voxel occupancy
        # Assuming voxel_texture stores occupancy or color; check for non-zero/non-transparent
        voxel_data = voxel_texture.load(voxel_cell)
        if voxel_data != 0: # Simplified occupancy check
            # Calculate global voxel index
            global_idx = voxel_cell.x + voxel_cell.y * brick_size + voxel_cell.z * brick_size * brick_size
            return global_idx
            
        # Advance Inner DDA
        min_t_v = min(t_next_v.x, min(t_next_v.y, t_next_v.z))
        axis_mask_v = step(t_next_v, min_t_v)
        
        t_next_v += axis_mask_v * t_delta_voxel
        voxel_cell += int3(axis_mask_v) * int3(sign(dir_local))
        
    return 0
```

## Subtopics

### Rasterized Brick AABB Depth Buffer
A hybrid rendering technique where camera-facing brick faces are rasterized into a depth buffer to determine the ray origin for DDA, allowing it to skip empty space much faster than DDA alone [S3, unofficial]. It differs from standard MultiDDA by using rasterization to cull the initial traversal start point rather than relying solely on the brick grid traversal [S3, unofficial]. Choose this when rendering dynamic voxel grids where the overhead of regenerating quads is acceptable for the performance gain [S3, unofficial]. It improves performance in large scenes (e.g., sponza-1024-1 FPS +28.99%) but slows down significantly when looking along a surface inside a brick, as DDA must still perform full brick + voxel traversal [S3, unofficial]. Pitfall: Quads are regenerated every frame to support dynamic voxel grids, adding overhead [S3, unofficial].

### Per-ray hierarchical-DDA
A GPU parallelism strategy where one thread per input point walks the active voxels along its ray and writes via lock-free atomic accumulation [S4, unofficial]. It differs from MultiDDA by focusing on the compute dispatch pattern for updates (e.g., LiDAR-TSDF) rather than just rendering [S4, unofficial]. Choose this for volumetric fusion or update kernels where atomic writes are required [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
-   Time: $O(N_b + N_v)$ in the worst case, where $N_b$ is the number of bricks traversed and $N_v$ is the number of voxels traversed within occupied bricks. In sparse scenes, this is significantly faster than naive DDA's $O(N_{total})$.
-   Memory: Requires storage for the brick grid texture ($N_b^3$ texels) and the voxel grid texture ($N_{voxels}$ texels). The brick grid adds a small overhead but enables significant skipping [S2, S3, unofficial].

**Measured Performance:**
-   In sparse scenes (e.g., sponza-256-2), MultiDDA provides massive gains: FPS +184.93%, Walltime -82.73% compared to naive DDA [S3, unofficial].
-   In dense scenes (e.g., sponza-256-1), the overhead of the two-level traversal results in minimal impact: FPS +0.04% [S3, unofficial].
-   With Rasterized Brick AABB Depth Buffer, performance uplifts are observed in large scenes: sponza-1024-1 FPS +28.99%, sponza-1024-2 FPS +36.64% [S3, unofficial].

**Strengths and Limitations:**
-   Strengths: Excellent performance in scenes with large empty spaces; simple implementation compared to octrees or BVHs for voxel data [S2, S3, unofficial].
-   Limitations: Poor performance with incoherent rays due to SIMD divergence when entering the inner traversal loop (not all lanes hit a brick at the same time) [S2, unofficial]. Overhead may not provide gains if the brick grid is not selective enough (e.g., dense scenes or loose brick sizes when viewing from afar) [S3, unofficial].

**When to Use:**
-   Use for real-time rendering of large, sparse voxel scenes where empty space skipping is critical.
-   Do not use for dense scenes or when memory bandwidth is the primary bottleneck, as the additional texture lookups for the brick grid may not be justified.

## Background
MultiDDA was developed as part of voxel rendering experiments to improve upon naive DDA traversal [S2, unofficial]. It was implemented in the VoxelRT project by dubiousconst282 [S2, unofficial]. The method gained further attention through devlogs by nelari, who implemented and benchmarked it against naive DDA and other acceleration structures [S3, unofficial]. It is a foundational technique for hierarchical voxel traversal, often serving as a precursor to more complex structures like eXtendedBrickMap (3-level grid) [S2, unofficial].

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S3] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
[S4] Dynamic GPU Sparse Grids and Voxel-sequential ..., https://research.nvidia.com/labs/prl/sifakis2026dynamic/dynamic_grids.pdf, fetched 2026-10-29 (unofficial)
