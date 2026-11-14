# Coherent and Connectivity Traversal

Also called: Coherent Ray Traversal, Connectivity-Aware Traversal

## Overview
This family accelerates ray-grid traversal by exploiting spatial coherence among ray bundles or by enforcing strict topological connectivity rules to resolve boundary ambiguities. Unlike standard cell-by-cell methods like [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) which process rays individually and may suffer from divergence or diagonal jumps, this family processes rays as coherent frustums or resolves tie-breaking conditions explicitly. It solves the problem of packet divergence in coherent ray tracing and normal inaccuracies in uniform grid traversal.

## Shared principles
All methods in this family build on [Ray-grid traversal](17-ray-grid-traversal.md), sharing the core mechanism of advancing analytically from one cell boundary to the next, avoiding fixed-distance sampling steps [S5][S7, unofficial]. The algorithm determines the next cell by comparing the distances to the upcoming boundaries on the x, y, and z axes, selecting the minimum distance to advance [S2, unofficial][S3, unofficial][S5].

### Mathematical Foundation
Let the ray be defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized or unnormalized) [G]. The grid is defined by cell boundaries. For a uniform grid with voxel size $v$ and dimensions $N_x, N_y, N_z$, the traversal calculates the distances to the next boundaries for each axis $i \in \{x, y, z\}$ [S2, unofficial][S3, unofficial][S5]:

1.  **Step Direction ($\text{step}_i$):** Determines the index increment direction [S4, unofficial].
    $$
    \text{step}_i = \begin{cases} 
    +1 & \text{if } d_i > 0 \\
    -1 & \text{if } d_i < 0 \\
    0 & \text{if } d_i = 0 
    \end{cases}
    $$
    If $d_i = 0$, the ray never crosses a boundary in that dimension, and the corresponding distance is treated as infinity [S5].

2.  **Time/Distance Delta ($\Delta t_i$ or $s_{\Delta i}$):** The distance along the ray to traverse one voxel width [S2, unofficial][S5].
    $$
    \Delta t_i = \frac{v_i}{|d_i|} \cdot \text{step}_i \quad \text{or} \quad s_{\Delta i} = \frac{\Delta x_i}{|\widehat{d}_i|}
    $$
    where $v_i$ is the voxel size and $\widehat{d}_i$ is the unit direction component [S2, unofficial][S5].

3.  **Time/Distance to Next Boundary ($t_{max, i}$ or $s_{max, i}$):** The distance from the ray origin to the first boundary crossed in axis $i$ [S2, unofficial][S3, unofficial][S5].
    $$
    t_{max, i} = \frac{\text{boundary}_i - o_i}{d_i}
    $$
    The initial boundary is determined by the starting voxel index [S3, unofficial]. If the ray starts inside the grid, the initial boundary is the next one in the direction of travel [S2, unofficial][S3, unofficial][S5].

### Traversal Loop
The core loop iterates while the ray remains within the grid bounds [S5]:
1.  Identify the axis $k$ with the minimum boundary distance: $t_{min} = \min(t_{max, x}, t_{max, y}, t_{max, z})$ [S2, unofficial][S3, unofficial][S5].
2.  Record the current cell and the interval $[t_{entry}, t_{min}]$ [S1, unofficial][S5].
3.  Advance the index in axis $k$: $\text{index}_k \leftarrow \text{index}_k + \text{step}_k$ [S2, unofficial][S3, unofficial][S5].
4.  Update the boundary distance for axis $k$: $t_{max, k} \leftarrow t_{max, k} + \Delta t_k$ [S2, unofficial][S3, unofficial][S5].
5.  Update $t_{entry} \leftarrow t_{min}$ [S5].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_traversal(ray_origin, ray_direction, voxel_size, grid_bounds):
    # Initialize current voxel index
    current_voxel = [int(ray_origin[i] // voxel_size) for i in range(3)]
    current_voxel = [max(grid_bounds.min[i], min(grid_bounds.max[i], current_voxel[i])) for i in range(3)]
    
    # Step direction per axis
    step = [1 if ray_direction[i] > 0 else (-1 if ray_direction[i] < 0 else 0) for i in range(3)]
    
    # Distance to next boundary
    next_boundary = [(current_voxel[i] + step[i]) * voxel_size for i in range(3)]
    t_max = [(next_boundary[i] - ray_origin[i]) / ray_direction[i] if ray_direction[i] != 0 else float('inf') for i in range(3)]
    
    # Distance delta per axis
    t_delta = [voxel_size / abs(ray_direction[i]) if ray_direction[i] != 0 else 0.0 for i in range(3)]
    
    t_entry = 0.0
    
    while grid_bounds.min[0] <= current_voxel[0] <= grid_bounds.max[0] and \
          grid_bounds.min[1] <= current_voxel[1] <= grid_bounds.max[1] and \
          grid_bounds.min[2] <= current_voxel[2] <= grid_bounds.max[2]:
        
        min_idx = t_max.index(min(t_max))
        t_exit = t_max[min_idx]
        
        yield current_voxel, t_entry, t_exit
        
        current_voxel[min_idx] += step[min_idx]
        t_max[min_idx] += t_delta[min_idx]
        t_entry = t_exit
```

## Subtopics

### [Coherent Grid Traversal](2987-coherent-grid-traversal.md)
Accelerates ray tracing by processing bundles of coherent rays as frustums, advancing slice-by-slice along a dominant axis rather than cell-by-cell. It solves the packet divergence problem inherent in cell-by-cell traversal (like 3DDDA), where rays in a group disagree on the next cell, by maintaining packet coherence through incremental frustum-slice overlap computation. Unlike hierarchical structures like kd-trees, CGT uses a flat uniform grid, enabling extremely fast per-frame rebuilds suitable for fully dynamic scenes.

### [Hero-like 4-Connectivity (H4T)](3436-hero-like-4-connectivity-h4t.md)
Guarantees strict 4-connectivity by explicitly resolving boundary crossing ambiguities via stable sorting of intersection distances. It solves the problem of diagonal jumps (8-connectivity errors) and associated normal inaccuracies that occur when a ray crosses an edge or corner in standard Digital Differential Analyzer (DDA) approaches. Unlike its sibling [Hierarchical Planes 4-Connectivity (HP4T)](3444-hierarchical-planes-4-connectivity-hp4t.md), which uses plane intersections and clamping, H4T relies on a 2x2 cell test and stable sorting to determine the first cell entered, achieving 100% correctness in randomized testing.

### Ray Connectivity
#### definition
- [S1, unofficial] A property defining the set of cells a ray can traverse to from its current cell, inspired by pixel connectivity in image processing.
#### formula
- [S1, unofficial] Taxicab metric distance between $P_1=(x_1, y_1)$ and $P_2=(x_2, y_2)$ is $|x_1 - x_2| + |y_1 - y_2|$.
- [S1, unofficial] Chebyshev metric distance between $P_1=(x_1, y_1)$ and $P_2=(x_2, y_2)$ is $\max(|x_1 - x_2|, |y_1 - y_2|)$.
#### number
- [S1, unofficial] In 2D, there are 4-connectivity (orthogonal neighbors) and 8-connectivity (including diagonal neighbors).

## Background
The concept of connectivity in ray traversal was formalized in a thesis by Hur [S1, unofficial], which analyzed the behavior of rays crossing voxel corners. The thesis noted that standard DDA methods might traverse through corners, leading to 8-connectivity, whereas specific algorithms aim for 4-connectivity to ensure correct normal calculation and topological consistency. The term "Coherent Grid Traversal" refers to methods that exploit the coherence of ray packets, a technique widely used in real-time ray tracing for dynamic scenes [S4, S5].

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
