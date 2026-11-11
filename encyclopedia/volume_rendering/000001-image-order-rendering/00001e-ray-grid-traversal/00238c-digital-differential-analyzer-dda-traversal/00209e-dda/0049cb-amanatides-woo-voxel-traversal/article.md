# Amanatides-Woo Voxel Traversal

Also called: Amanatides-Woo, Amanatides and Woo, A Fast Voxel Traversal Algorithm for Ray Tracing, AW87

## Overview

The Amanatides-Woo algorithm is a fast, branch-minimal method for traversing a uniform 3D voxel grid along a ray, visiting every cell intersected by the ray in front-to-back order [S13, S31, S38, unofficial]. It solves the problem of efficiently locating geometry in a dense grid without performing expensive per-voxel intersection tests or suffering from the gaps and redundancies of naive fixed-step ray marching [S31, S41, unofficial]. Unlike [Siddon's Algorithm](2590-siddon-s-algorithm.md), which computes exact parametric intersection lengths for integration, Amanatides-Woo steps discretely through grid cells using incremental comparisons, making it ideal for real-time ray casting and shadow calculation [S1, unofficial][S25]. It sets itself apart from [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md) by using floating-point $t$-values for boundary distances rather than integer error accumulation, allowing it to handle arbitrary ray directions and non-unit grid spacing more naturally [S3, S31, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
1.  **Ray:** Defined by origin $O = (o_x, o_y, o_z)$ and direction $D = (d_x, d_y, d_z)$ [S1, S30, unofficial]. The direction vector is typically normalized, but the algorithm works with any non-zero direction [S1, S30, unofficial].
2.  **Grid:** A uniform 3D voxel grid defined by:
    *   `grid_min`: World-space minimum corner $(min_x, min_y, min_z)$ [S1, S13, unofficial].
    *   `grid_max`: World-space maximum corner $(max_x, max_y, max_z)$ [S1, S13, unofficial].
    *   `dims`: Grid dimensions $(N_x, N_y, N_z)$ or a single side length for cubic grids [S1, S13, unofficial].
    *   Alternatively, `voxel_size` can be provided if the grid is isotropic and aligned to the origin, but the general case uses bounds [S1, S63, unofficial].

**Outputs:**
*   A sequence of voxel indices $(i, j, k)$ visited by the ray, in order of increasing distance from the origin [S56, unofficial].
*   Optionally, the parametric distance $t$ at which the ray enters each voxel, or the specific hit point if a solid voxel is found [S1, S46, unofficial].

### Data Structures
The algorithm maintains several 3D vectors (or arrays of size 3) for the current state:
1.  **`voxel_pos`**: `ivec3` containing the current integer voxel coordinates $(i, j, k)$ [S23, unofficial].
2.  **`step`**: `ivec3` containing the direction of traversal along each axis ($+1$, $-1$, or $0$) [S39, unofficial].
3.  **`t_max`**: `vec3` containing the parametric distance $t$ along the ray to the next grid boundary crossing for each axis [S31, unofficial].
4.  **`t_delta`**: `vec3` containing the parametric distance required to traverse one full voxel width along each axis [S44, unofficial].
5.  **`entry_t`**: `float` representing the parametric distance from the ray origin to the entry point of the grid AABB [S1, S13, unofficial].

### Numbered Steps

1.  **Ray-AABB Intersection (Clipping):**
    Determine if the ray intersects the grid's bounding box [S50, unofficial]. Compute the entry and exit parametric distances $t_{entry}$ and $t_{exit}$ for the grid [S1, S14, unofficial]. If the ray misses the grid, terminate [S1, S14, unofficial].
    *   For each axis $a \in \{x, y, z\}$:
        *   $t_1 = (min_a - o_a) / d_a$
        *   $t_2 = (max_a - o_a) / d_a$
        *   $t_{min, a} = \min(t_1, t_2)$
        *   $t_{max, a} = \max(t_1, t_2)$
    *   $t_{entry} = \max(t_{min, x}, t_{min, y}, t_{min, z})$ [S1, S26, unofficial].
    *   $t_{exit} = \min(t_{max, x}, t_{max, y}, t_{max, z})$ [S1, S26, unofficial].
    *   If $t_{min} > t_{max}$, the ray misses the grid [S1, S23, unofficial].
    *   *Note:* If the ray origin is inside the grid, $t_{entry}$ may be negative [S23, unofficial]. The algorithm typically clamps the start of traversal to $t_{entry}$ or a small epsilon offset to avoid boundary precision issues [S1, S63, unofficial].

2.  **Initialization:**
    *   **Calculate Voxel Size:** If not given, $V_a = (max_a - min_a) / N_a$ [G]. For unit voxels, $V_a=1$ [S1, S61, unofficial].
    *   **Determine Step Direction:** For each axis $a$, if $d_a \ge 0$, $step_a = 1$; else $step_a = -1$ [S77, unofficial]. Handle $d_a = 0$ by setting $step_a = 0$ and ensuring $t_{delta, a} = \infty$ [S31, unofficial].
    *   **Calculate Inverse Direction:** $inv\_d_a = 1.0 / d_a$ [G]. Clamp to a large value if $d_a \approx 0$ to avoid division by zero [S40, S65, unofficial].
    *   **Calculate $t_{delta}$:** The distance along the ray to cross one voxel. $t_{delta, a} = |inv\_d_a| \times V_a$ [S26, unofficial]. If $d_a = 0$, $t_{delta, a} = \infty$ [S13, S31, S63, unofficial].
    *   **Determine Starting Voxel:** Calculate the voxel index containing the ray entry point [S13, unofficial].
        *   $entry\_pos = O + D \times (t_{entry} + \epsilon)$, where $\epsilon$ is a small value (e.g., $10^{-4}$) to nudge the point inside the grid [S1, S63, unofficial].
        *   $voxel\_pos_a = \lfloor (entry\_pos_a - min_a) / V_a \rfloor$ [S1, S63, unofficial].
        *   Clamp $voxel\_pos$ to $[0, N_a-1]$ [S1, S63, unofficial].
    *   **Calculate Initial $t_{max}$:** The parametric distance to the *next* boundary crossing for each axis [S20, unofficial].
        *   $t_{max, a} = (voxel\_pos_a - entry\_pos_a + \max(step_a, 0)) \times inv\_d_a$ [S1, unofficial].

3.  **Traversal Loop:**
    Repeat until a solid voxel is found or the ray exits the grid ($t > t_{exit}$):
    *   **Check Voxel:** Query the voxel at $voxel\_pos$. If solid, return the hit information (index, normal, $t$) [S1, S46, unofficial].
    *   **Find Minimum Axis:** Identify the axis $a$ with the smallest $t_{max, a}$. This is the next boundary the ray crosses [S1, S13, unofficial].
    *   **Step:**
        *   $voxel\_pos_a += step_a$
        *   $t_{max, a} += t_{delta, a}$
    *   **Boundary Check:** If $voxel\_pos_a$ is outside $[0, N_a-1]$, terminate (ray exited grid) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def amanatides_woo_traverse(ray_origin, ray_dir, grid_min, grid_max, dims):
    """
    Traverses a uniform 3D voxel grid along a ray.
    
    Args:
        ray_origin: (float, float, float) Ray origin O
        ray_dir: (float, float, float) Ray direction D (non-zero)
        grid_min: (float, float, float) Grid minimum corner
        grid_max: (float, float, float) Grid maximum corner
        dims: (int, int, int) Number of voxels (Nx, Ny, Nz)
        
    Returns:
        List of (voxel_index, t_entry) tuples, or empty list if miss.
    """
    
    # 1. Ray-AABB Intersection
    t_min = 0.0
    t_max = float('inf')
    
    for axis in range(3):
        if abs(ray_dir[axis]) < 1e-10:
            if ray_origin[axis] < grid_min[axis] or ray_origin[axis] > grid_max[axis]:
                return [] # Miss
        else:
            t1 = (grid_min[axis] - ray_origin[axis]) / ray_dir[axis]
            t2 = (grid_max[axis] - ray_origin[axis]) / ray_dir[axis]
            
            axis_t_min = min(t1, t2)
            axis_t_max = max(t1, t2)
            
            t_min = max(t_min, axis_t_min)
            t_max = min(t_max, axis_t_max)
            
    if t_min > t_max:
        return [] # Miss
    
    epsilon = 1e-4
    entry_t = t_min + epsilon
    entry_pos = (
        ray_origin[0] + ray_dir[0] * entry_t,
        ray_origin[1] + ray_dir[1] * entry_t,
        ray_origin[2] + ray_dir[2] * entry_t
    )
    
    # 2. Initialization
    voxel_size = (
        (grid_max[0] - grid_min[0]) / dims[0],
        (grid_max[1] - grid_min[1]) / dims[1],
        (grid_max[2] - grid_min[2]) / dims[2]
    )
    
    inv_dir = (
        1.0 / ray_dir[0] if abs(ray_dir[0]) > 1e-10 else float('inf'),
        1.0 / ray_dir[1] if abs(ray_dir[1]) > 1e-10 else float('inf'),
        1.0 / ray_dir[2] if abs(ray_dir[2]) > 1e-10 else float('inf')
    )
    
    step = (
        1 if ray_dir[0] >= 0 else -1,
        1 if ray_dir[1] >= 0 else -1,
        1 if ray_dir[2] >= 0 else -1
    )
    
    t_delta = (
        abs(inv_dir[0]) * voxel_size[0],
        abs(inv_dir[1]) * voxel_size[1],
        abs(inv_dir[2]) * voxel_size[2]
    )
    
    voxel_pos = (
        int((entry_pos[0] - grid_min[0]) / voxel_size[0]),
        int((entry_pos[1] - grid_min[1]) / voxel_size[1]),
        int((entry_pos[2] - grid_min[2]) / voxel_size[2])
    )
    
    voxel_pos = (
        max(0, min(voxel_pos[0], dims[0] - 1)),
        max(0, min(voxel_pos[1], dims[1] - 1)),
        max(0, min(voxel_pos[2], dims[2] - 1))
    )
    
    # Initialize t_max for each axis using S1's formula
    t_max_x = (voxel_pos[0] - entry_pos[0] + (1 if step[0] > 0 else 0)) * inv_dir[0] if abs(inv_dir[0]) != float('inf') else float('inf')
    t_max_y = (voxel_pos[1] - entry_pos[1] + (1 if step[1] > 0 else 0)) * inv_dir[1] if abs(inv_dir[1]) != float('inf') else float('inf')
    t_max_z = (voxel_pos[2] - entry_pos[2] + (1 if step[2] > 0 else 0)) * inv_dir[2] if abs(inv_dir[2]) != float('inf') else float('inf')
    
    # 3. Traversal Loop
    visited = []
    current_t = entry_t
    
    while True:
        if not (0 <= voxel_pos[0] < dims[0] and 
                0 <= voxel_pos[1] < dims[1] and 
                0 <= voxel_pos[2] < dims[2]):
            break
            
        visited.append((voxel_pos, current_t))
        
        # Find axis with minimum t_max
        if t_max_x < t_max_y:
            if t_max_x < t_max_z:
                voxel_pos = (voxel_pos[0] + step[0], voxel_pos[1], voxel_pos[2])
                t_max_x += t_delta[0]
                current_t += t_max_x - t_delta[0] # Update current t
            else:
                voxel_pos = (voxel_pos[0], voxel_pos[1], voxel_pos[2] + step[2])
                t_max_z += t_delta[2]
                current_t += t_max_z - t_delta[2]
        else:
            if t_max_y < t_max_z:
                voxel_pos = (voxel_pos[0], voxel_pos[1] + step[1], voxel_pos[2])
                t_max_y += t_delta[1]
                current_t += t_max_y - t_delta[1]
            else:
                voxel_pos = (voxel_pos[0], voxel_pos[1], voxel_pos[2] + step[2])
                t_max_z += t_delta[2]
                current_t += t_max_z - t_delta[2]
                
        if not (0 <= voxel_pos[0] < dims[0] and 
                0 <= voxel_pos[1] < dims[1] and 
                0 <= voxel_pos[2] < dims[2]):
            break
            
    return visited
```

### Parameters and Edge Cases
*   **Zero Direction Component:** If $d_a = 0$, the ray is parallel to the grid planes of axis $a$. $t_{delta, a}$ and $t_{max, a}$ should be set to $\infty$ so this axis is never selected for stepping [S31, S40, unofficial].
*   **Origin Inside Grid:** If the ray origin is inside the grid, $t_{entry}$ is negative. The algorithm must start traversal from the voxel containing the origin, not from $t_{entry}$ [S1, S50, unofficial].
*   **Precision:** Floating-point errors can cause the ray to skip voxels or visit them twice if $t_{max}$ values are very close. Using a small epsilon when calculating the entry position helps [S1, S63, unofficial].
*   **Degenerate Input:** If the ray direction is zero, the algorithm is undefined. Check for $|D| < \epsilon$ and return a miss [S53, unofficial].

## Subtopics

### Worst-Case Iteration Analysis
The worst-case number of iterations occurs when the ray travels along the diagonal through exclusively empty voxels. The number of steps is proportional to $N_x + N_y + N_z$, where $N_i$ is the grid dimension in axis $i$. This visits every intersected voxel exactly once [S78, S11, unofficial].

### [DDA Skip](5260-dda-skip.md)
A variant that skips empty voxels in bulk. It uses the standard DDA to step through the grid but checks an occupancy flag. If a voxel is empty, it may skip ahead or continue stepping, depending on the acceleration structure (e.g., octree) used to determine the next occupied region.

### [Occupancy-aware DDA Traversal](5261-occupancy-aware-dda-traversal.md)
Similar to DDA Skip, this variant samples every voxel but branches based on occupancy. If a voxel is active, it records samples; if inactive, it skips time. This is useful for volumetric rendering where empty space can be skipped.

### Vectorized Amanatides and Woo Variant
An elegant variant using vector operations and operator overloading to avoid explicit branching for axis selection. It computes the step direction and boundary distances using element-wise vector math, finding the minimum axis via `argmin`. This simplifies the code and can be more efficient on SIMD hardware [S21, unofficial].

### DDA with Root Voxel Check
A variant that checks for a specific 'root' voxel (e.g., 0,0,0) during traversal. This is used for debugging or specific logical conditions where hitting the origin triggers a state change or early termination [S2, unofficial].

### Shadow Casting
A specific application where the Amanatides-Woo traversal is used to cast shadow rays from surface points to light sources. It traverses the voxel grid to determine if any opaque voxels block the light, computing hard shadows efficiently [S37, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(N_x + N_y + N_z)$ per ray, where $N_i$ is the number of voxels intersected along axis $i$. This visits every intersected voxel exactly once [S11, S31, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space. The algorithm only requires a few 3D vectors for state, regardless of grid size [S1, S13, unofficial].
*   **Performance:** Extremely fast due to minimal floating-point operations per step (one comparison, one addition, one increment) [S38, unofficial]. It avoids expensive division operations in the main loop by precomputing inverse directions [S13, unofficial].
*   **Strengths:**
    *   No per-voxel intersection tests.
    *   Guarantees no voxels are skipped or visited twice (for non-zero direction components) [S31, S52, unofficial].
    *   Simple to implement and highly parallelizable [S8, unofficial].
*   **Limitations:**
    *   Requires a uniform grid. Non-uniform grids require hierarchical structures [S3, S78, unofficial].
    *   Floating-point precision issues can arise in very large grids or with rays nearly parallel to grid planes [S31, unofficial].
    *   If the ray misses the grid, the initial AABB test is required [S1, unofficial].
*   **When to Use:** Real-time voxel raycasting, shadow mapping in voxel scenes, and any scenario requiring efficient traversal of a uniform 3D grid [S1, S37, unofficial].
*   **When Not to Use:** Non-uniform grids (use [Siddon's Algorithm](2590-siddon-s-algorithm.md) or hierarchical structures), or when exact integration lengths are required for scientific visualization [S1, unofficial][S25].

## Background

The Amanatides-Woo algorithm was introduced by Amanatides and Woo in their 1987 paper "A Fast Voxel Traversal Algorithm for Ray Tracing" [S13, S36, unofficial]. It is an extension of the 2D DDA line drawing algorithm (Bresenham's algorithm) to 3D. The algorithm was designed to efficiently traverse a 3D grid of voxels for ray tracing, avoiding the computational cost of intersecting every voxel with the ray. It has become a standard technique in voxel-based rendering, including games like Minecraft and voxel-based medical imaging [S1, unofficial][S12][S43, unofficial].

## Sources

[S1] MΛX - Amanatides and Woo's fast Voxel Traversal, https://m4xc.dev/articles/amanatides-and-woo/, fetched 2026-10-24 (unofficial)
[S2] sebbooth/vkEngine, https://github.com/sebbooth/vkEngine, fetched 2026-10-24 (unofficial)
[S3] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S4] Raytracing Intersections: Voxel Traversal | GelamiSalami, https://gelamisalami.github.io/blog/posts/voxel-traversal/, fetched 2026-10-24 (unofficial)
[S5] Casting Rays: Voxel Rendering Using Discrete Ray Tracing, https://castingrays.blogspot.com/2014/01/voxel-rendering-using-discrete-ray.html, fetched 2026-10-24 (unofficial)
[S6] knosvoxel/voxel_algorithm, https://github.com/knosvoxel/voxel_algorithm, fetched 2026-10-24 (unofficial)
[S7] AThilenius/fast-voxel-traversal-rs, https://github.com/AThilenius/fast-voxel-traversal-rs, fetched 2026-10-24 (unofficial)
[S8] junpei-kariyazono/gpu-voxel-tracer, https://github.com/junpei-kariyazono/gpu-voxel-tracer, fetched 2026-10-24 (unofficial)
[S9] Bálint's blog, https://blog.balintcsala.com/posts/voxel-tracing/, fetched 2026-10-24 (unofficial)
[S10] JackEvans24/voxel-traversal, https://github.com/JackEvans24/voxel-traversal, fetched 2026-10-24 (unofficial)
[S11] Noah-Everett/Fast-Voxel-Traversal-Python, https://github.com/Noah-Everett/Fast-Voxel-Traversal-Python, fetched 2026-10-24 (unofficial)
[S12] 3D Raycasting - Voxel.Wiki, https://voxel.wiki/wiki/raycasting/, fetched 2026-11-01
[S13] A Fast Voxel Traversal Algorithm, http://cse.yorku.ca/~amana/research/grid.pdf, fetched 2026-10-24 (unofficial)
[S14] cgyurgyik/fast-voxel-traversal-algorithm, https://github.com/cgyurgyik/fast-voxel-traversal-algorithm, fetched 2026-10-24 (unofficial)
[S15] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S16] feelinfine/tracer, https://github.com/feelinfine/tracer, fetched 2026-10-24 (unofficial)
[S17] DJayalath/RayTracingEngine, https://github.com/DJayalath/RayTracingEngine, fetched 2026-10-24 (unofficial)
[S18] SirJonthe/VoxelRayTrace, https://github.com/SirJonthe/VoxelRayTrace, fetched 2026-10-24 (unofficial)
[S19] Implementation of voxel traversal algorithm from "A Fast Voxel Traversal Algorithm for Ray Tracing" (Amanatides and Woo) · GitHub, https://gist.github.com/garymacindoe/895430c1e53a6e50cb35, fetched 2026-10-24 (unofficial)
[S20] bonsairobo/grid-ray-rs, https://github.com/bonsairobo/grid-ray-rs, fetched 2026-10-24 (unofficial)
[S21] Cast ray to select block in voxel game, https://gamedev.stackexchange.com/questions/47362, fetched 2026-10-25 (unofficial)
[S22] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
[S23] tacf/cpu-voxel-raytracer-raylib, https://github.com/tacf/cpu-voxel-raytracer-raylib, fetched 2026-10-24 (unofficial)
[S24] jason-s-yu/fast-voxel-traversal, https://github.com/jason-s-yu/fast-voxel-traversal, fetched 2026-10-24 (unofficial)
[S25] Fredrik2002/2D-Fast-Voxel-Traversal-Algorithm, https://github.com/Fredrik2002/2D-Fast-Voxel-Traversal-Algorithm, fetched 2026-10-24
[S26] Musta-Kissa/quadtree-raycast, https://github.com/Musta-Kissa/quadtree-raycast, fetched 2026-10-24 (unofficial)
[S27] A Voxel Renderer for Learning C/C++ – Jacco's Blog, https://jacco.ompf2.com/2021/02/01/a-voxel-renderer-for-learning-c-c/, fetched 2026-10-24 (unofficial)
[S28] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S29] Fast grid-based ray tracing in C++ and Python - GitHub, https://github.com/acschaefer/ray_tracing, fetched 2026-10-25 (unofficial)
[S30] fast-voxel-traversal-algorithm /overview - GitHub, https://github.com/cgyurgyik/fast-voxel-traversal-algorithm/blob/master/overview/FastVoxelTraversalOverview.md, fetched 2026-10-25 (unofficial)
[S31] Amanatides-Woo Voxel Traversal | 3D DDA Learning Guide, https://edukatesengkang.com/2026/08/30/how-to-learn-amanatides-woo-fast-voxel-traversal-3d-dda-tmax-tdelta-grid-crossings-and-robust-ray-marching/, fetched 2026-10-25 (unofficial)
[S32] Fast Voxel Traversal Algorithm - GitHub, https://github.com/taigama/Fast-Voxel-Traversal-Algorithm, fetched 2026-10-25 (unofficial)
[S33] Fast Voxel Traversal Algorithm for Ray Tracing, https://fpwrasmussen.github.io/DTU-Special-Course/src/shadow/fast_voxel_traversal.html, fetched 2026-10-25
[S34] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S35] Line Rasterization and Grid Traversal, https://trzy.org/line_rasterization/, fetched 2026-10-25 (unofficial)
[S36] Fast Voxel Traversal for Ray Tracing | PDF, https://www.scribd.com/document/6913705/A-faster-voxel-traversal-algorithm-for-ray-tracing, fetched 2026-10-25 (unofficial)
[S37] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S38] Fast Voxel Traversal for Ray Tracing, https://www.scribd.com/document/841721013/Voxel-Traversal-Paper, fetched 2026-10-25 (unofficial)
[S39] A Fast Voxel Traversal Algorithm for Ray Tracing · GitHub, https://gist.github.com/Nik--/0beb2f5f2b93182e467db86223f6e1d2, fetched 2026-10-25 (unofficial)
[S40] GitHub - DeadlockCode/voxel_ray_traversal, https://github.com/DeadlockCode/voxel_ray_traversal, fetched 2026-10-25 (unofficial)
[S41] Ray-Marched Voxel Simulation - Joris Rijsdijk, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S42] kit: ray traversal, graph analysis, bin packing, hex A* — round 7, https://github.com/shizukutanaka/IZANAGI/pull/23, fetched 2026-10-25 (unofficial)
[S43] DEV-015: 3-D voxel grid with ray/voxel DDA traversal (Stage 3), https://github.com/e0404/autonomous-ion-mc/pull/45, fetched 2026-10-25 (unofficial)
[S44] 【Ray tracing】Voxelを作ってみる, https://qiita.com/misaki_mofu/items/026cd353054856921995, fetched 2026-10-25 (unofficial)
[S45] 体渲染中的体素遍历技术:从理论到实践-百度开发者中心, https://developer.baidu.com/article/detail.html?id=6975905, fetched 2026-10-25 (unofficial)
[S46] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
[S47] [3D TUI] DDA voxel raycaster core, https://github.com/cybrid-systems/aura/issues/1983, fetched 2026-10-25 (unofficial)
[S48] feat(raytrace): parallel-beam voxel ray tracer for transmission imaging, https://github.com/ornlneutronimaging/NEREIDS/issues/456, fetched 2026-10-25 (unofficial)
[S49] Digital Differential Analyzer (DDA) Algorithm for 3D Grid/Voxels, https://github.com/bitshifter/glam-rs/issues/563, fetched 2026-10-25 (unofficial)
[S50] 光线追踪小记:空间加速结构Regular Grid与3DDDA, https://zhuanlan.zhihu.com/p/415869768, fetched 2026-10-25 (unofficial)
[S51] 在glam-rs中实现3D网格的DDA算法 - AtomGit | GitCode博客, https://blog.gitcode.com/8503a870cc78647d71b758940e1d905c.html, fetched 2026-10-25 (unofficial)
[S52] Voxel DDA Traversal Diagram, https://www.shadertoy.com/view/MXfGRj, fetched 2026-10-26 (unofficial)
[S53] feat(renderer): Amanatides–Woo DDA voxel raycast ( #1983 / #1979 ), https://github.com/cybrid-systems/aura/commit/885aac266adebc723b418803faa697415185b1eb, fetched 2026-10-25 (unofficial)
[S54] Tracé des voxels sur le passage d'un rayon, http://raphaello.univ-fcomte.fr/ig/raytracing/ExemplesGLUt/LigneVoxels.htm, fetched 2026-10-25 (unofficial)
[S55] Tracé des voxels sur le passage d'un rayon, http://nicolas.janey.free.fr/RayTracing/Exemples/LigneVoxels.htm, fetched 2026-10-25 (unofficial)
[S56] Voxel Traversal Algorithm (Ray Casting) - GameDev.net, https://gamedev.net/blogs/entry/2265248-voxel-traversal-algorithm-ray-casting/, fetched 2026-10-25 (unofficial)
[S57] A Fast Voxel Traversal AlgorithmforRay Tracing, https://devpress.csdn.net/v1/article/detail/102420463, fetched 2026-10-26 (unofficial)
[S58] java3d dda算法,A Fast Voxel Traversal Algorithm for Ray Tracing, https://blog.csdn.net/weixin_29013129/article/details/116040448, fetched 2026-10-26 (unofficial)
[S59] 医疗影像开发实战:如何用光线投射算法快速生成DRR图像(附Python代码), https://un.csdn.net/98nhkcg5t04h, fetched 2026-10-26 (unofficial)
[S60] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S61] nerfstudio-project/nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
[S62] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S63] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S64] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S65] michschw/GpuPro3Vox, https://github.com/michschw/GpuPro3Vox, fetched 2026-10-26 (unofficial)
[S67] urielfederez/Volume-Data-Visualization, https://github.com/urielfederez/Volume-Data-Visualization, fetched 2026-10-27 (unofficial)
[S68] lolleko/voxelscape, https://github.com/lolleko/voxelscape, fetched 2026-10-27 (unofficial)
[S69] gabkhanfig/Cube, https://github.com/gabkhanfig/Cube, fetched 2026-10-27 (unofficial)
[S70] GitHub - Arlorean/RaymarchVoxels: Render a voxel model in Unity by ..., https://github.com/Arlorean/RaymarchVoxels, fetched 2026-10-27
[S71] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S72] d3x0r/DualMarchingTetrahedra, https://github.com/d3x0r/DualMarchingTetrahedra, fetched 2026-10-27 (unofficial)
[S73] feat(#40): Find the voxel at the screen center with a voxel ray, https://github.com/nicolas-goyon/Elysian-Outpost/pull/113, fetched 2026-10-27 (unofficial)
[S74] wlim33/vox-ocean, https://github.com/wlim33/vox-ocean, fetched 2026-10-27 (unofficial)
[S75] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S76] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
[S77] GitHub - ernerdgn/voxel-engine: A voxel engine written from scratch in ..., https://github.com/ernerdgn/voxel-engine, fetched 2026-10-29 (unofficial)
[S78] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S79] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
