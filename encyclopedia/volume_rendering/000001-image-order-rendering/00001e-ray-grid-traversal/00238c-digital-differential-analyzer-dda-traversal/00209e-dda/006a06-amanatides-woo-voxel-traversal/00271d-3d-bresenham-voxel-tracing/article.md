# 3D Bresenham Voxel Tracing

Also called: 3D line scan-conversion, Fast Voxel Traversal Algorithm for Ray Tracing, Voxel DDA Ray Marching, voxelDDA

## Overview
3D Bresenham Voxel Tracing is a discrete ray traversal algorithm that approximates a continuous ray as a sequence of adjacent voxels in a uniform 3D grid using integer arithmetic [S3, S5, unofficial]. It solves the problem of efficiently identifying which voxels a ray intersects without computing floating-point ray-AABB intersections at every step [S1, S4, unofficial]. It is distinct from analytical slab-intersection methods by relying on integer error accumulation to step through adjacent voxels [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\vec{o}$, ray direction $\vec{d}$, grid bounds (minimum and maximum coordinates), voxel size $s$ (typically $s=1$), and a voxel grid data structure [S1, S6, S9, unofficial].
*   **Output:** A sequence of voxel coordinates $(x, y, z)$ visited by the ray, or a hit result containing the first occupied voxel's index, normal, and distance [S6, S9, unofficial].

**Data Structures:**
*   **Voxel Grid:** A dense 1D array or 3D array. For a grid of size $W \times H \times D$, the linear index for coordinate $(x, y, z)$ in row-major order is:
    $$ \text{index} = (z \cdot H + y) \cdot W + x $$
    [S2, unofficial]
*   **State Variables:** Current voxel coordinates $(x, y, z)$, step directions $(x_{inc}, y_{inc}, z_{inc}) \in \{-1, 1\}$, and error accumulators $(c_1, c_2)$ [S3, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Determine the entry point $t_0$ where the ray enters the grid's Axis-Aligned Bounding Box (AABB) [S1, S9, unofficial]. If the ray does not intersect the grid, terminate [S9, unofficial].
    *   Calculate the initial world position: $\vec{f}_0 = \vec{o} + t_0 \cdot \vec{d}$ [S1, unofficial].
    *   Compute the initial voxel index $\vec{i}_0 = \lfloor \vec{f}_0 \rfloor$ (component-wise floor) [S1, unofficial]. This is the starting voxel $(x, y, z)$.
    *   Calculate the target end position $\vec{f}_1$ corresponding to the exit of the grid or a maximum distance [S1, unofficial].
    *   Compute deltas: $\Delta x = |x_f - x_i|$, $\Delta y = |y_f - y_i|$, $\Delta z = |z_f - z_i|$ [S3, unofficial].
    *   Determine step directions: $x_{inc} = \text{sign}(x_f - x_i)$, $y_{inc} = \text{sign}(y_f - y_i)$, $z_{inc} = \text{sign}(z_f - z_i)$ [S3, unofficial].
    *   Identify the dominant axis (the axis with the largest delta). Let this be $D_{dom}$ with value $\Delta_{dom}$ [S3, unofficial].
    *   Initialize error accumulators $c_1 = c_2 = \Delta_{dom} / 2$ [S3, unofficial]. This centers the error term to handle the start voxel correctly [S3, unofficial].

2.  **Traversal Loop:**
    *   Iterate $i$ from 1 to $\Delta_{dom}$ (inclusive) [S3, unofficial].
    *   **Step on Dominant Axis:** Increment the coordinate on the dominant axis by its step direction ($x \leftarrow x + x_{inc}$ if X is dominant, etc.) [S3, unofficial].
    *   **Update Error Accumulators:**
        *   If X is dominant: $c_1 \leftarrow c_1 + \Delta y$, $c_2 \leftarrow c_2 + \Delta z$ [S3, unofficial].
        *   If Y is dominant: $c_1 \leftarrow c_1 + \Delta x$, $c_2 \leftarrow c_2 + \Delta z$ [S3, unofficial].
        *   If Z is dominant: $c_1 \leftarrow c_1 + \Delta y$, $c_2 \leftarrow c_2 + \Delta x$ [S3, unofficial].
    *   **Check Non-Dominant Axes:**
        *   If $c_1 \ge \Delta_{dom}$:
            *   $c_1 \leftarrow c_1 - \Delta_{dom}$
            *   Increment the corresponding non-dominant coordinate by its step direction [S3, unofficial].
        *   If $c_2 \ge \Delta_{dom}$:
            *   $c_2 \leftarrow c_2 - \Delta_{dom}$
            *   Increment the other non-dominant coordinate by its step direction [S3, unofficial].
    *   **Process Voxel:** The current $(x, y, z)$ is a voxel intersected by the ray. Check occupancy or perform shading [S3, S6, unofficial].
    *   **Boundary Check:** If the updated coordinates exceed grid bounds $[0, W-1] \times [0, H-1] \times [0, D-1]$, terminate [S9, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bresenham_voxel_tracing(ray_origin, ray_direction, grid_min, grid_max, voxel_size=1.0):
    # 1. Slab intersection to find t_entry and t_exit
    inv_dir = [1.0 / d if abs(d) > 1e-9 else float('inf') for d in ray_direction]
    t0 = [(grid_min[i] - ray_origin[i]) * inv_dir[i] for i in range(3)]
    t1 = [(grid_max[i] - ray_origin[i]) * inv_dir[i] for i in range(3)]
    t_entry = max(min(t0), min(t1))
    t_exit = min(max(t0), max(t1))
    if t_entry > t_exit or t_exit < 0:
        return # No intersection
    # 2. Map to voxel indices with epsilon offset to avoid boundary precision issues
    f0 = [ray_origin[i] + t_entry * ray_direction[i] + 1e-4 * ray_direction[i] for i in range(3)]
    f1 = [ray_origin[i] + t_exit * ray_direction[i] for i in range(3)]
    xi = int(f0[0] // voxel_size)
    yi = int(f0[1] // voxel_size)
    zi = int(f0[2] // voxel_size)
    xf = int(f1[0] // voxel_size)
    yf = int(f1[1] // voxel_size)
    zf = int(f1[2] // voxel_size)
    # Clamp to grid bounds
    xi = max(0, min(xi, int(grid_max[0] // voxel_size) - 1))
    yi = max(0, min(yi, int(grid_max[1] // voxel_size) - 1))
    zi = max(0, min(zi, int(grid_max[2] // voxel_size) - 1))
    xf = max(0, min(xf, int(grid_max[0] // voxel_size) - 1))
    yf = max(0, min(yf, int(grid_max[1] // voxel_size) - 1))
    zf = max(0, min(zf, int(grid_max[2] // voxel_size) - 1))

    dx = xf - xi
    dy = yf - yi
    dz = zf - zi
    xinc = 1 if dx > 0 else -1
    yinc = 1 if dy > 0 else -1
    zinc = 1 if dz > 0 else -1
    abs_dx = abs(dx)
    abs_dy = abs(dy)
    abs_dz = abs(dz)

    # 3. Identify dominant axis and initialize error accumulators
    c1 = 0
    c2 = 0
    yield (xi, yi, zi)

    if abs_dx >= abs_dy and abs_dx >= abs_dz:
        c1 = abs_dx // 2
        c2 = abs_dx // 2
        for _ in range(abs_dx):
            xi += xinc
            c1 += abs_dy
            if c1 >= abs_dx:
                c1 -= abs_dx
                yi += yinc
            c2 += abs_dz
            if c2 >= abs_dz:
                c2 -= abs_dz
                zi += zinc
            yield (xi, yi, zi)
    elif abs_dy >= abs_dx and abs_dy >= abs_dz:
        c1 = abs_dy // 2
        c2 = abs_dy // 2
        for _ in range(abs_dy):
            yi += yinc
            c1 += abs_dx
            if c1 >= abs_dy:
                c1 -= abs_dy
                xi += xinc
            c2 += abs_dz
            if c2 >= abs_dz:
                c2 -= abs_dz
                zi += zinc
            yield (xi, yi, zi)
    else:
        c1 = abs_dz // 2
        c2 = abs_dz // 2
        for _ in range(abs_dz):
            zi += zinc
            c1 += abs_dy
            if c1 >= abs_dz:
                c1 -= abs_dz
                yi += yinc
            c2 += abs_dx
            if c2 >= abs_dx:
                c2 -= abs_dx
                xi += xinc
            yield (xi, yi, zi)
```

**Parameters:**
*   `voxel_size`: Typically 1.0 to eliminate division in coordinate mapping [S1, unofficial].
*   `max_steps`: Optional limit to prevent infinite loops in degenerate cases (e.g., ray parallel to grid face but outside bounds) [S9, unofficial]. Default 128 for a 64x64x64 grid [S9, unofficial].

**Edge Cases and Pitfalls:**
*   **Axis-Aligned Rays:** If a ray direction component is zero, the standard DDA division by zero is avoided in Bresenham's integer logic. However, the dominant axis selection must handle ties (e.g., if $dx=0, dy=0, dz>0$, Z is dominant) [S3, S8, unofficial].
*   **Boundary Conditions:** The algorithm must clamp coordinates to $[0, \text{dim}-1]$ to avoid out-of-bounds memory access [S9, unofficial].
*   **Connectivity:** The standard Bresenham algorithm produces a 6-connected path in 3D (moving along one axis at a time, or diagonally across faces). A 26-connected path (allowing corner crossings) may miss intersections that a 6-connected path detects, but requires less computation [S5, unofficial]. The basic integer accumulation described above yields a path that is topologically consistent with the line segment but may not visit every voxel touched by the continuous ray if the ray passes through a voxel corner [S5, unofficial].
*   **Precision:** Using integer arithmetic avoids floating-point accumulation errors, but the initial mapping from world to voxel coordinates must be precise [S1, unofficial].

## Subtopics

### [Integer-based Amanatides and Woo Voxel Traversal](4246-integer-based-amanatides-and-woo-voxel-traversal.md)
This method implements the Amanatides and Woo voxel traversal algorithm using scaled integer arithmetic to determine the sequence of grid cells intersected by a ray, ensuring exactness and stability by avoiding floating-point operations. It serves as the core traversal loop for image-order volume rendering or ray casting.

### [Voxel Traversal with Rounding Error Correction](6207-voxel-traversal-with-rounding-error-correction.md)
Voxel Traversal with Rounding Error Correction is a method for stepping a ray through a voxel grid that explicitly compensates for floating-point precision errors in position calculations. It builds on [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md) by adding a correction step to ensure the ray enters the correct adjacent voxel, preventing misses or incorrect entries caused by rounding artifacts.

### [Liu Extension of Bresenham](12811-liu-extension-of-bresenham.md)
The Liu Extension of Bresenham is a voxel traversal algorithm that extends the classic 2D Bresenham line algorithm to 3D grid traversal by detecting and visiting diagonal neighbors when the ray intersects a cell corner. Unlike standard Bresenham which visits only one neighbor per step, this method checks if the error exceeds a specific threshold to decide whether to visit both $(x+1, y)$ and $(x+1, y+1)$ in 2D, or the corresponding diagonal paths in 3D. It is distinguished from its siblings by its reliance on a geometric intersection check against a derived threshold rather than simple integer error accumulation, offering performance advantages even on modern hardware where floating-point operations are efficient.

### [Zalik et al. Algorithm](12816-zalik-et-al-algorithm.md)
The Zalik et al. algorithm is a voxel traversal method that corrects the topological incompleteness of standard 3D Bresenham tracing by ensuring all cells intersected by a ray are visited. It differs from its parent [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md) by adding a post-processing phase to recover missed cells, whereas siblings like [Integer-based Amanatides and Woo Voxel Traversal](4246-integer-based-amanatides-and-woo-voxel-traversal.md) compute intersections analytically.

### [BCC/FCC Ray Tracing](13311-bcc-fcc-ray-tracing.md)
BCC/FCC Ray Tracing is a volume rendering and object representation method that samples 3-D data on Body Centered Cubic (BCC) or Face Centered Cubic (FCC) grids instead of standard cubic grids. It solves topological defects inherent in cubic grids—specifically, the possibility of a ray passing through an object surface without intersecting any voxel—by utilizing grids with superior topological properties that guarantee closed surfaces and robust discrete curve/surface definitions. Unlike cubic grid methods such as [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md) or [Integer-based Amanatides and Woo Voxel Traversal](4246-integer-based-amanatides-and-woo-voxel-traversal.md), this approach defines voxels as Voronoi cells (Truncated Octahedra for BCC, Rhombic Dodecahedra for FCC) and traces rays by minimizing the cross-sectional area of the ray beam within these polyhedral structures.

### [Template-Based Volume Viewing](15404-template-based-volume-viewing.md)
Template-Based Volume Viewing is a ray-grid traversal technique for parallel projection that exploits inter-ray coherence by precomputing a single sequence of voxel increments (a template) and reusing it for all rays, thereby avoiding per-ray line algorithm activation. It solves the discretization errors and performance overhead inherent in naive per-ray traversal, where varying initial positions cause voxels to be skipped or visited multiple times. Unlike siblings that recompute traversal logic per ray, this method ensures all rays share the same form regardless of length, distributing discretization error uniformly.

### [Ray-Grid Boundary Intersection for First Voxel Index](18141-ray-grid-boundary-intersection-for-first-voxel-index.md)
This method computes the initial voxel index for ray traversal by determining the exact point where a ray enters a uniform axis-aligned grid and converting that point to grid coordinates. It differs from its siblings, such as [Integer-based Amanatides and Woo Voxel Traversal](4246-integer-based-amanatides-and-woo-voxel-traversal.md) and [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md), by explicitly solving for the entry intersection using parametric line equations rather than relying on incremental stepping or integer arithmetic approximations. This approach is chosen when an exact geometric entry point is required before initiating a traversal algorithm, particularly for handling arbitrary ray orientations without special-casing vertical or horizontal components early in the pipeline.

### Voxel-Based Lighting
A lighting calculation performed using the normal and position derived from the voxel traversal hit, rather than the continuous surface intersection of standard ray tracing [S10, unofficial]. It calculates the real-world position of the hit using the traversal distance `hitT` (`vec3 realPos = ro + hitT * rd;`), computes diffuse lighting intensity using the dot product of the hit normal and light position, applies ambient and diffuse scaling, and mixes between shadow and cube colors based on intensity [S10, unofficial]. Standard ambient occlusion and shadow calculations may not work correctly because they are computed on the continuous geometry rather than the voxel grid [S10, unofficial].

### Stochastic Ray Bouncing
A shading technique used within the voxel tracer where rays bounce off voxel surfaces [S11, unofficial]. It uses a probabilistic mix of perfect reflection and diffuse scattering to simulate indirect lighting and color bleeding, limited to a maximum of 3 bounces [S11, unofficial]. If a random number `rand > 0.9`, a perfect specular reflection is performed; otherwise, a random diffuse direction is generated, reflected if it points into the surface, and attenuated by the voxel's color [S11, unofficial]. The diffuse reflection correction is $dir_{out} = dir_{rand} - 2 \cdot (dir_{rand} \cdot normal) \cdot normal$ if $dir_{rand} \cdot normal < 0$ [S11, unofficial]. The demo has very limited denoising, resulting in extremely noisy images [S11, unofficial].

### Integer Line Voxelization
An integer-based line voxelization algorithm evaluated for its runtime and approximation error relative to RLV [S12, unofficial]. ILV was 20-250% slower than RLV in runtime evaluations [S12, unofficial]. The approximation error of ILV, measured by Jaccard distance, generally started at 20% and grew up towards 25% for higher voxel resolutions [S12, unofficial].

### Real Line Voxelization
An algorithm for optimal scanline voxelization of 3D models that relies on line voxelization, evaluated as the best performing method among those compared [S12, unofficial]. RLV performed better in every case compared to ILV and Bresenham; ILV was 20-250% slower and Bresenham was 20-500% slower [S12, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of voxels traversed, bounded by the grid dimension in the worst case ($O(\max(W, H, D))$) [S9, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space for the traversal state, plus $O(W \cdot H \cdot D)$ for the voxel grid [S2, unofficial].
*   **Strengths:** No division by zero issues for axis-aligned rays; simple integer logic; low memory overhead; easy to implement [S3, S5, unofficial].
*   **Limitations:** May miss voxels intersected by the ray (topological incompleteness) if the path is not 26-connected or if error accumulation is not corrected [S5, unofficial]. Less precise than floating-point DDA for determining exact hit positions [S12, unofficial].

## Background
The method originates from the classic Bresenham line algorithm, extended to 3D for voxelization and raycasting [S3, S5, unofficial]. It is often referred to as "Voxel DDA Ray Marching" or "3D line scan-conversion" [S5, S6, unofficial]. It is used in voxel-based games (e.g., Minecraft-like engines) for collision detection and raycasting [S4, S8, unofficial], and in medical volume rendering as a discrete approximation of ray casting [S5, unofficial]. The "Fast Voxel Traversal Algorithm for Ray Tracing" by Amanatides and Woo (1987) is a related but distinct floating-point method; Bresenham's integer variant is often used as a simpler alternative or as a basis for integer-based optimizations [S1, S5, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Prototype the portable compute-ray data and traversal baseline, https://github.com/PetrSeifert/voxel-nexus/issues/52, fetched 2026-10-25 (unofficial)
[S3] Tracé des voxels sur le passage d'un rayon, http://raphaello.univ-fcomte.fr/ig/raytracing/ExemplesGLUt/LigneVoxels.htm, fetched 2026-10-25 (unofficial)
[S4] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S5] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S6] Divine-Star-Software/DivineVoxelEngine, https://github.com/Divine-Star-Software/DivineVoxelEngine, fetched 2026-10-26 (unofficial)
[S7] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S8] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
[S9] an1pse/voxel-traversal, https://github.com/an1pse/voxel-traversal, fetched 2026-10-27 (unofficial)
[S10] 【Ray tracing】Voxelを作ってみる, https://qiita.com/misaki_mofu/items/026cd353054856921995, fetched 2026-10-25 (unofficial)
[S11] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
[S12] A Comparison of Optimal Scanline Voxelization Algorithms, http://urn.kb.se/resolve?urn=urn:nbn:se:liu:diva-168331, fetched 2026-10-24 (unofficial)
