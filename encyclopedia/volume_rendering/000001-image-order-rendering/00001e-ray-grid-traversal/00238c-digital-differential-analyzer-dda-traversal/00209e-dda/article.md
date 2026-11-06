# DDA

Also called: DDA Algorithm, DDA Raycasting Algorithm, Delta Calculation Helper, Digital Differential Analyzer

## Overview
The Digital Differential Analyzer (DDA) is a grid traversal algorithm that determines the sequence of uniform grid cells (voxels or tiles) intersected by a ray, outputting the indices of visited cells in front-to-back order [S3, S4, S7, unofficial]. It solves the problem of efficiently enumerating grid intersections without missing thin cells or performing redundant work, which plagues naive ray marching [S2]. Unlike [Amanatides and Woo Voxel Traversal](3132-amanatides-and-woo-voxel-traversal.md) which explicitly computes t-values for grid boundaries, the DDA formulation often operates by maintaining and comparing accumulated distances to the next grid line in each dimension, stepping the minimum distance at each iteration [S3, S4, S7, unofficial]. It is distinguished by its suitability for branchless GPU implementation, where conditional logic is replaced by vector masks and arithmetic updates [S3, S10, unofficial].

## How it works
### Inputs and outputs
**Inputs:**
*   `ray_origin`: 3D vector $(o_x, o_y, o_z)$ specifying the ray start point [S4, S7, S10, unofficial].
*   `ray_direction`: 3D vector $(d_x, d_y, d_z)$ specifying the ray direction. Source [S10, unofficial] states the direction does not need to be normalized, while [S4, unofficial] requires normalization; the algorithm works with unnormalized directions by using reciprocal components for scaling [S10, unofficial].
*   `grid_bounds`: Optional bounding box $(grid_{min}, grid_{max})$ or resolution $grid_{res}$ to limit traversal [S10, unofficial].
*   `max_time`: Maximum parameter $t$ along the ray to traverse [S2][S9, unofficial].

**Outputs:**
*   A sequence of integer voxel coordinates $(i, j, k)$ visited by the ray [S2][S3, unofficial].
*   For each step, the time $t$ of entry into the current cell and the exit time into the next cell [S7, S10, unofficial].

### Data structures
The DDA state maintains fields that vary by implementation. The branchless GPU variant maintains [S3, unofficial]:
*   `pos`: Integer vector $(i, j, k)$ representing the current voxel index [S3, unofficial].
*   `side_dist`: Floating-point vector representing the distance along the ray to the next grid boundary in each dimension [S3, unofficial].
*   `delta_dist`: Floating-point vector representing the distance along the ray to cross one full grid cell in each dimension (i.e., $1/|d_x|, 1/|d_y|, 1/|d_z|$) [S3, unofficial].
*   `step_dir`: Integer vector $(\pm 1, \pm 1, \pm 1)$ indicating the direction of increment for each axis [S3, unofficial].
OpenVDB's `DDA` class uses `mVoxel`, `mNext`, `mDelta`, `mStep`, `mT0`, and `mT1` instead [S8, unofficial].

### Numbered steps
1.  **Initialization**:
    *   Compute the initial voxel index: $pos = \lfloor ray\_origin \rfloor$ [S3, S4, unofficial].
    *   Compute the reciprocal direction components: $rcp\_d = 1 / ray\_direction$ [S10, unofficial]. Handle zero-direction components by setting their reciprocal to a large magnitude to avoid division by zero [S10, S11, unofficial].
    *   Compute `delta_dist` as the absolute value of the reciprocal direction: $delta\_dist = |rcp\_d|$ [S3, S10, unofficial].
    *   Compute `step_dir` based on the sign of the ray direction: $step\_dir = \text{sign}(ray\_direction)$ [S3, S10, unofficial].
    *   Compute the fractional position within the current voxel: $p_0 = ray\_origin - pos$ [S10, unofficial].
    *   Compute the initial distance to the next grid boundary for each axis [S4, S10, unofficial]. If $d_x > 0$, the next boundary is at $x = pos_x + 1$, so the distance is $(1 - p_{0,x}) \cdot |rcp_{d,x}|$ [S10, unofficial]. If $d_x < 0$, the next boundary is at $x = pos_x$, so the distance is $-p_{0,x} \cdot |rcp_{d,x}|$ [S4, unofficial]. This can be unified as:
        $$ side\_dist = (step\_dir \cdot (pos - ray\_origin) + 0.5 \cdot step\_dir + 0.5) \cdot delta\_dist $$
        Note: The specific formula in [S3, unofficial] uses a heuristic offset; [S10, unofficial] derives it from geometric intersection. Both aim to calculate the distance to the immediate next grid line [S3, S10, unofficial].
    *   (Optional) Clip `pos` to grid bounds if the ray starts outside [S10, unofficial].

2.  **Stepping Loop**:
    *   While within grid bounds and traversal is active:
        *   Invoke a callback or process the current voxel at `pos` [S10, unofficial].
        *   Determine which axis to step next by finding the axis with the minimum `side_dist` [S3, unofficial]. Create a boolean mask `mask` where `mask[i]` is true if `side_dist[i]` is less than or equal to the other components (cyclically compared for tie-breaking) [S3, unofficial]:
            $$ mask = (side\_dist \le \min(side\_dist.yzx, side\_dist.zxy)) $$
        *   Update `side_dist` by adding `delta_dist` to the components indicated by `mask` [S3, unofficial].
        *   Update `pos` by adding `step_dir` to the components indicated by `mask` [S3, unofficial].
        *   (Optional) Check for termination conditions (e.g., hit detected, bounds exceeded, max_time reached) [S4, unofficial][S6].

### Formulas
*   **Distance to next grid line (X-axis, $d_x > 0$)**:
    $$ t_x = \frac{\lceil o_x \rceil - o_x}{d_x} $$
    Subsequent crossings are spaced by $1/d_x$ [S7, unofficial].
*   **Scaling factor for step length**: The length of the ray segment corresponding to a unit step in the x-axis is $\sqrt{1 + (d_y/d_x)^2}$ [S5, unofficial].
*   **Safe Inverse**: To handle near-axis-aligned rays without division by zero:
    $$ \text{safe\_inv}(x) = \begin{cases} \text{select}(-10^{20}, 10^{20}, x \ge 0) & \text{if } |x| < 10^{-8} \\ 1/x & \text{otherwise} \end{cases} $$
    [S11, unofficial].

### Parameters
*   `Dim`: In some implementations (e.g., OpenVDB's HDDA/DDA), a template parameter defining the step size or tile dimension, defaulting to 1 for standard voxel traversal [S2][S8, unofficial].
*   `startTime`, `maxTime`: Bounds for the ray parameter $t$ [S2][S9, unofficial].

### Edge cases and pitfalls
*   **Zero Direction Components**: If a ray component is zero, the ray is parallel to the grid lines in that dimension. The reciprocal is infinite, and the distance to the next grid line is infinite, effectively disabling stepping in that axis [S10, S11, unofficial].
*   **Ray on Grid Boundary**: If the ray origin lies exactly on a grid line, the initial distance calculation may yield zero. A guard `dt > 0` prevents processing zero-length segments or double-counting cells [S10, S7, unofficial].
*   **GPU Branching**: Standard implementations using `if` statements for axis selection cause branch divergence on GPUs. The branchless mask-based approach avoids this penalty [S3, unofficial].
*   **Negative Directions**: The initial distance calculation must account for the direction of travel. If $d_x < 0$, the next boundary is at the lower integer coordinate, not the upper [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_traverse(ray_origin, ray_direction, grid_res, max_time, callback):
    # 1. Initialization
    # Compute reciprocal direction, handling zeros safely [S11, unofficial]
    rcp_d = vec3(1.0)
    for i in range(3):
        if abs(ray_direction[i]) < 1e-8:
            rcp_d[i] = 1e20 if ray_direction[i] >= 0 else -1e20
        else:
            rcp_d[i] = 1.0 / ray_direction[i]

    abs_rcp_d = abs(rcp_d)
    pos = floor(ray_origin)
    step_dir = sign(ray_direction)
    p0 = ray_origin - vec3(floor(ray_origin))

    # Initial distances to next grid boundaries [S4, S10, unofficial]
    side_dist = vec3(0.0)
    for i in range(3):
        if ray_direction[i] > 0:
            side_dist[i] = (1.0 - p0[i]) * abs_rcp_d[i]
        elif ray_direction[i] < 0:
            side_dist[i] = -p0[i] * abs_rcp_d[i]
        else:
            side_dist[i] = 1e20

    delta_dist = abs_rcp_d

    # 2. Stepping Loop
    while True:
        # Process current voxel [S10, unofficial]
        callback(pos)

        # Find axis with minimum side_dist [S3, unofficial]
        min_val = min(side_dist)
        # Cyclic comparison for tie-breaking [S3, unofficial]
        mask = vec3(side_dist <= min(side_dist.yzx, side_dist.zxy))

        # Update distances and position [S3, unofficial]
        side_dist += mask * delta_dist
        pos += mask * step_dir

        # Check bounds and termination [S10, unofficial]
        if not (0 <= pos.x < grid_res.x and 0 <= pos.y < grid_res.y and 0 <= pos.z < grid_res.z):
            break
        # Optional: break if max_time reached based on accumulated distance [S10, unofficial]

    return pos
```

## Subtopics

### [Amanatides–Woo Voxel Traversal](1068-amanatides-woo-voxel-traversal.md)
The Amanatides–Woo algorithm is the canonical DDA variant for uniform grids. It explicitly computes the t-values for crossing grid boundaries in each dimension, merges them into a sorted stream, and steps through the grid. It is preferred when exact t-values are needed for shading or when the grid is not unit-sized. It avoids the pitfalls of naive marching by jumping directly to the next boundary [1068-amanatides-woo-voxel-traversal.md].

### [TRFRay3 (DDA Ray Representation)](2104-trfray3-dda-ray-representation.md)
A specific implementation of DDA used in the TRFRay3 library, focusing on efficient ray representation and traversal for volume rendering. It optimizes the DDA state for cache locality and SIMD execution [2104-trfray3-dda-ray-representation.md].

### [Differentiable DDA Loop](2474-differentiable-dda-loop.md)
A variant of the DDA loop designed for differentiable rendering. It traverses the grid while maintaining the computational graph for backpropagation, allowing gradients to flow through the traversal process itself. This is crucial for optimizing scene parameters or camera poses in neural rendering [2474-differentiable-dda-loop.md].

### [Multi-level DDA with Adaptive Mip Selection](4086-multi-level-dda-with-adaptive-mip-selection.md)
This variant uses DDA to step through a voxel grid but leverages multiple mip levels of a majorant texture. It dynamically selects the mip level at each step to determine the step size, balancing performance (larger steps) and accuracy (smaller steps) without branching [4086-multi-level-dda-with-adaptive-mip-selection.md].

## Operating characteristics
*   **Time Complexity**: $O(K)$, where $K$ is the number of visited cells. The algorithm visits each intersected cell exactly once [S2].
*   **Memory Complexity**: $O(1)$ auxiliary space, storing only the current state (position, distances, deltas) [S3, S8, unofficial].
*   **Performance**: Highly efficient on GPUs due to branchless execution. The mask-based update avoids divergence [S3, S10, unofficial].
*   **Strengths**: Simple to implement, robust, uses integer steps for position updates to avoid floating-point error accumulation, suitable for real-time raycasting [S3, unofficial][S6].
*   **Limitations**: Requires uniform grid spacing. Non-uniform grids require modifications or hierarchical variants like [Hierarchical DDA with Super-brick Skipping](5197-hierarchical-dda-with-super-brick-skipping.md) [S8, unofficial].
*   **When to use**: Uniform voxel grids, real-time raycasting, GPU shaders [S6][S10, unofficial].
*   **When not to use**: Non-uniform grids, hierarchical structures (use HDDA or VDB traversal), or when exact t-values are required for complex intersection tests (use Amanatides-Woo) [S8, S10, unofficial].

## Background
The Digital Differential Analyzer (DDA) algorithm originated as a line-drawing algorithm in computer graphics, attributed to Bresenham's contemporaries, but adapted for ray tracing and voxel traversal in the 1990s and 2000s [S3, unofficial][S6][S7, S8, unofficial]. It is widely used in voxel engines (e.g., Wolfenstein 3D-style raycasting) and modern volume rendering [S3, unofficial][S6]. The branchless GPU implementation has been popularized by shader-based tutorials and libraries like OpenVDB's nanovdb [S3, unofficial][S6][S7, S8, unofficial].

## Sources
[S1] sebbooth/vkEngine, https://github.com/sebbooth/vkEngine, fetched 2026-10-24 (unofficial)
[S2] OpenVDB: DDA< RayT, CoordT, Dim > Class Template Reference, https://academysoftwarefoundation.github.io/openvdb/classnanovdb_1_1DDA.html, fetched 2026-10-24
[S3] The DDA algorithm | Voxel Raytracing, https://ii-day-ii.github.io/voxel_raytracer/2023/05/05/the-dda-algorithm.html, fetched 2026-10-24 (unofficial)
[S4] The DDA Algorithm, https://aaaa.sh/creatures/dda-algorithm-interactive/, fetched 2026-10-24 (unofficial)
[S5] Super Fast Ray Casting in Tiled Worlds using DDA, https://www.youtube.com/watch?v=NbSee-XM7WA, fetched 2026-10-25 (unofficial)
[S6] Raycasting Algorithm | dderevjanik/ts-raycasting | DeepWiki, https://deepwiki.com/dderevjanik/ts-raycasting/4.1-raycasting-algorithm, fetched 2026-10-25
[S7] The DDA Algorithm, explained interactively, https://news.ycombinator.com/item?id=43543007, fetched 2026-10-30 (unofficial)
[S8] openvdb/nanovdb/nanovdb/math/HDDA.h at master, https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/math/HDDA.h, fetched 2026-10-25 (unofficial)
[S9] OpenVDB: openvdb/math/DDA.h Source File, https://www.openvdb.org/documentation/doxygen/DDA_8h_source.html, fetched 2026-10-26 (unofficial)
[S10] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
[S11] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S12] mithraskuipers/InteractiveLearning-Raycasting, https://github.com/mithraskuipers/InteractiveLearning-Raycasting, fetched 2026-10-24 (unofficial)
