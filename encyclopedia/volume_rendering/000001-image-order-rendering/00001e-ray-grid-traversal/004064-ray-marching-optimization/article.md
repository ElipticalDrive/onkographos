# Ray Marching Optimization
## Overview
Ray marching optimization refers to techniques that accelerate volume rendering by replacing fixed-distance sampling steps with analytical traversal of voxel boundaries or by using auxiliary data structures to skip empty space [S1]. Unlike standard fixed-step ray marching, which samples at uniform intervals regardless of scene content, optimized methods advance the ray analytically from one cell boundary to the next, minimizing floating-point operations and redundant samples [G]. This family builds on [Ray-grid traversal](17-ray-grid-traversal.md) by focusing on the efficiency of the traversal loop and the integration of sampling strategies [G].

## Shared principles
All methods in this family share the core mechanism of advancing analytically from one cell boundary to the next, avoiding fixed-distance sampling steps [G]. The algorithm determines the next cell by comparing the distances to the upcoming boundaries on the x, y, and z axes, selecting the minimum distance to advance [G].

### Mathematical Foundation
Let the ray be defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized or unnormalized) [G]. The grid is defined by cell boundaries. For a uniform grid with voxel size $v$ and dimensions $N_x, N_y, N_z$, the traversal relies on three key precomputed values per axis $i \in \{x, y, z\}$ [G]:

1.  **Step Direction ($\text{step}_i$):** Determines the index increment direction [G].
    $$
    \text{step}_i = \begin{cases} 
    +1 & \text{if } d_i > 0 \\
    -1 & \text{if } d_i < 0 \\
    0 & \text{if } d_i = 0 
    \end{cases}
    $$
    If $d_i = 0$, the ray never crosses a boundary in that dimension, and the corresponding distance is treated as infinity [G].

2.  **Time/Distance Delta ($\Delta t_i$):** The distance along the ray to traverse one voxel width [G].
    $$
    \Delta t_i = \frac{v_i}{|d_i|}
    $$
    where $v_i$ is the voxel size [G].

3.  **Time/Distance to Next Boundary ($t_{max, i}$):** The distance from the ray origin to the first boundary crossed in axis $i$ [G].
    $$
    t_{max, i} = \frac{\text{boundary}_i - o_i}{d_i}
    $$
    The initial boundary is determined by the starting voxel index [G]. If the ray starts inside the grid, the initial boundary is the next one in the direction of travel [G].

### Traversal Loop
The core loop iterates while the ray remains within the grid bounds [G]:
1.  Identify the axis $k$ with the minimum boundary distance: $t_{min} = \min(t_{max, x}, t_{max, y}, t_{max, z})$ [G].
2.  Record the current cell and the interval $[t_{entry}, t_{min}]$ [G].
3.  Advance the index in axis $k$: $\text{index}_k \leftarrow \text{index}_k + \text{step}_k$ [G].
4.  Update the boundary distance for axis $k$: $t_{max, k} \leftarrow t_{max, k} + \Delta t_k$ [G].
5.  Update $t_{entry} \leftarrow t_{min}$ [G].

When the ray crosses an edge or corner (i.e., two or three $t_{max}$ values are equal), the traversal must advance all corresponding axes to maintain 4-connectivity (or 6/8-connectivity in 2D/3D) and avoid zero-length segments [G]. The implementation uses independent `if` checks for ties rather than `if/elif` chains [G]. A tie tolerance $\varepsilon_{\mathrm{DDA}}$ is used to handle floating-point drift, typically defined as $64 \varepsilon_{\mathrm{machine}} \max(1 \text{ cm}, L_{\mathrm{grid}})$ [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_traversal(ray_origin, ray_direction, voxel_size, bounds_min, bounds_max):
    current_voxel = [int(ray_origin[i] // voxel_size) for i in range(3)]
    current_voxel = [max(bounds_min, min(bounds_max, v)) for v in current_voxel]
    step = [1 if ray_direction[i] > 0 else (-1 if ray_direction[i] < 0 else 0) for i in range(3)]
    next_boundary = [(current_voxel[i] + step[i]) * voxel_size for i in range(3)]
    t_max = [(next_boundary[i] - ray_origin[i]) / ray_direction[i] if ray_direction[i] != 0 else float('inf') for i in range(3)]
    t_delta = [voxel_size / abs(ray_direction[i]) if ray_direction[i] != 0 else float('inf') for i in range(3)]
    visited = []
    while all(bounds_min <= current_voxel[i] <= bounds_max for i in range(3)):
        visited.append(current_voxel[:])
        axis = t_max.index(min(t_max))
        current_voxel[axis] += step[axis]
        t_max[axis] += t_delta[axis]
    return visited
```

## Subtopics

### [Voxel Ray Marching with Traversal Order Optimization](1251-voxel-ray-marching-with-traversal-order-optimization.md)
Renders a 3D voxel texture by marching rays from the camera through the volume, compositing samples with front-to-back alpha blending to visualize the volume. It differs from standard fixed-step ray marching by optimizing the traversal order of voxel boundaries using precomputed time deltas ($t_{\Delta}$) and maximum boundary times ($t_{max}$), minimizing floating-point operations per step. The algorithm advances analytically from one cell boundary to the next, avoiding fixed-distance sampling steps. Choose this method when rendering dense voxel textures where minimizing per-step arithmetic overhead is critical.

### [Tiled Ray Marching with Grid Sampling](2586-tiled-ray-marching-with-grid-sampling.md)
Renders a single view of a 3D volume by generating rays, sampling density via grid sampling, and accumulating attenuation. It supports tiling to manage memory usage for large images. It differs from [Voxel Ray Marching with Traversal Order Optimization](1251-voxel-ray-marching-with-traversal-order-optimization.md) by using dense grid sampling (e.g., `grid_sample`) rather than sparse voxel traversal, and by explicitly tiling the image plane to bound peak memory usage. Choose this method when rendering large volumes on hardware with limited memory or when using differentiable rendering pipelines.

### Indicator Grid Ray Marching
**Definition**
A binary map cached and updated during training to signify empty areas, allowing for early termination of ray marching based on transmittance, thereby optimizing ray marching efficiency [S1].

**Step**
1. Initialize the indicator grid as fully occupied, assuming all space is dense [S1].
2. Sample a small batch of spatiotemporal points and pass them through the tiny MLP $\theta$ to obtain density [S1].
3. Update the indicator grid based on the obtained density distribution [S1].

**Comparison**
Compared to the original Lerplane [51] which uses a sample-net requiring extensive sampling, this method reduces unnecessary point sampling, enabling approximately 5 times faster inference (1.73 fps vs. 0.38 fps) [S1].

## Background
Originating from the need to accelerate volume rendering in medical imaging and real-time applications, ray marching optimization evolved from basic Digital Differential Analyzer (DDA) traversal [G]. The integration of indicator grids for early termination was introduced in the context of efficient deformable tissue reconstruction, specifically in the Lerplane framework [S1]. This approach was developed to address the high computational cost of surface-focused sampling in dynamic scenes, leveraging the observation that empty space can be identified and skipped during training and inference [S1].

## Sources
[S1] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
