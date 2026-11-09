# Digital Differential Analyzer (DDA) Traversal

Also called: 3D DDA Ray Traversal, Amanatides & Woo, DDA Algorithm, Digital Differential Analyzer

## Overview
Digital Differential Analyzer (DDA) Traversal is a grid traversal algorithm that determines the sequence of cells (voxels or tiles) intersected by a ray in a uniform, axis-aligned grid [S4, unofficial][S5, unofficial]. It solves the problem of efficiently stepping through a discrete spatial structure without performing expensive intersection tests or fixed-step ray marching, by calculating the parameterized distance ($t$) to the next grid boundary in each dimension and advancing along the axis with the smallest $t$ [S7, unofficial][S5, unofficial]. It sets itself apart from naive fixed-step marching by using boundary-crossing logic to guarantee no voxels are missed, and from hierarchical methods by operating directly on a flat uniform grid [S1][S3, unofficial].

## How it works
The algorithm advances analytically from one cell boundary to the next. It determines the next cell by comparing the distances to the upcoming boundaries on the x, y, and z axes, selecting the minimum distance to advance [S2, unofficial][S3, unofficial][S5, unofficial].

### Inputs and Outputs
**Inputs:**
*   Ray origin $\mathbf{o} = (o_x, o_y, o_z)$ [G].
*   Ray direction $\mathbf{d} = (d_x, d_y, d_z)$ [G].
*   Voxel size $v = (v_x, v_y, v_z)$ [G].
*   Grid bounds: minimum index $\mathbf{I}_{min}$ and maximum index $\mathbf{I}_{max}$ [G].

**Outputs:**
*   A sequence of voxel indices $\mathbf{i} = (i_x, i_y, i_z)$ visited by the ray.
*   Optionally, the entry and exit distances $t_{entry}$ and $t_{exit}$ for each voxel [S1][S5, unofficial].

### Data Structures
The algorithm maintains the following state variables per axis $k \in \{x, y, z\}$:
1.  **Current Voxel Index ($\mathbf{i}$):** Integer coordinates of the current cell.
2.  **Step Direction ($\mathbf{s}$):** Integer vector $\{-1, 0, 1\}$ indicating the direction of index increment.
3.  **Time to Next Boundary ($t_{max}$):** Floating-point distance from the ray origin to the next grid plane boundary in each axis.
4.  **Time Delta ($t_{delta}$):** Floating-point distance along the ray to traverse one voxel width in each axis.

### Initialization
1.  **Compute Current Voxel:** Determine the starting voxel index by flooring the origin coordinates divided by voxel sizes [S2, unofficial]:
    $$
    i_k = \lfloor o_k / v_k \rfloor
    $$
    Clamp $\mathbf{i}$ to the grid bounds $[\mathbf{I}_{min}, \mathbf{I}_{max}]$ [G].
2.  **Compute Step Direction:** Determine the direction of traversal for each axis based on the ray direction sign [S4, unofficial]:
    $$
    s_k = \begin{cases} 
    +1 & \text{if } d_k > 0 \\
    -1 & \text{if } d_k < 0 \\
    0 & \text{if } d_k = 0 
    \end{cases}
    $$
3.  **Compute Time Delta:** Calculate the distance along the ray to cross one voxel [S2, unofficial][S5, unofficial]:
    $$
    t_{delta, k} = \frac{v_k}{|d_k|}
    $$
    If $d_k = 0$, set $t_{delta, k} = 0$ [S3, unofficial].
4.  **Compute Time to Next Boundary:** Calculate the distance from the ray origin to the first boundary crossed in axis $k$ [S2, unofficial][S3, unofficial][S5, unofficial]. The boundary coordinate is determined by the current voxel index and step direction [S3, unofficial]:
    $$
    t_{max, k} = \frac{(i_k + s_k) v_k - o_k}{d_k}
    $$
    If $d_k = 0$, set $t_{max, k} = \infty$ [S5, unofficial].

### Traversal Loop
Iterate while the current voxel $\mathbf{i}$ is within the grid bounds $[\mathbf{I}_{min}, \mathbf{I}_{max}]$ [S5, unofficial]:

1.  **Identify Axis:** Find the axis $k$ with the minimum $t_{max}$ value:
    $$
    k = \arg\min_{j \in \{x,y,z\}} (t_{max, j})
    $$
    In the case of coincident crossings (edge/corner), a fixed priority order (e.g., X, then Y, then Z) is used to break ties [S2, unofficial].
2.  **Record Cell:** Yield or process the current voxel $\mathbf{i}$ and the interval $[t_{entry}, t_{max, k}]$ [S1][S5, unofficial].
3.  **Advance Index:** Increment the index in the selected axis:
    $$
    i_k \leftarrow i_k + s_k
    $$
4.  **Update Boundary:** Update the boundary distance for the selected axis:
    $$
    t_{max, k} \leftarrow t_{max, k} + t_{delta, k}
    $$
5.  **Update Entry:** Set $t_{entry} \leftarrow t_{max, k}$ (the previous exit point) [S5, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_traversal(ray_origin, ray_direction, voxel_size, grid_min, grid_max):
    # 1. Initialize current voxel index
    # Floor division for integer indices
    current_voxel = [int(o // v) for o, v in zip(ray_origin, voxel_size)]
    
    # Clamp to grid bounds [G]
    for k in range(3):
        if current_voxel[k] < grid_min[k]:
            current_voxel[k] = grid_min[k]
        elif current_voxel[k] > grid_max[k]:
            current_voxel[k] = grid_max[k]
            
    # 2. Step direction per axis
    step = []
    for d in ray_direction:
        if d > 0:
            step.append(1)
        elif d < 0:
            step.append(-1)
        else:
            step.append(0)
            
    # 3. Distance delta per axis
    t_delta = []
    for d, v in zip(ray_direction, voxel_size):
        if d == 0:
            t_delta.append(0)
        else:
            t_delta.append(v / abs(d))
            
    # 4. Distance to next boundary
    # Boundary coordinate is (current_voxel + step) * voxel_size
    t_max = []
    for k in range(3):
        d = ray_direction[k]
        if d == 0:
            t_max.append(float('inf'))
        else:
            boundary_coord = (current_voxel[k] + step[k]) * voxel_size[k]
            t_max.append((boundary_coord - ray_origin[k]) / d)
            
    t_entry = 0.0
    
    # 5. Traversal Loop
    while True:
        # Check bounds
        if not (all(current_voxel[k] >= grid_min[k] for k in range(3)) and 
                all(current_voxel[k] <= grid_max[k] for k in range(3))):
            break
            
        # Find minimum boundary distance
        min_idx = 0
        min_val = t_max[0]
        for k in range(1, 3):
            if t_max[k] < min_val:
                min_val = t_max[k]
                min_idx = k
            # Tie-breaking: fixed priority X, Y, Z is implicit in strict < comparison 
            # if initialized in order, or explicit logic if needed [S2, unofficial]
        
        t_exit = min_val
        
        # Record cell and interval
        yield current_voxel, t_entry, t_exit
        
        # Advance index and update boundary
        current_voxel[min_idx] += step[min_idx]
        t_max[min_idx] += t_delta[min_idx]
        t_entry = t_exit
```

### Edge Cases and Pitfalls
*   **Zero Direction Components:** If $d_k = 0$, the ray never crosses a boundary in that dimension. The corresponding $t_{max}$ is set to infinity, and $t_{delta}$ is zero, effectively disabling stepping in that axis [S5, unofficial][S3, unofficial].
*   **Origin on Boundary:** If the ray origin lies exactly on a grid boundary $k \cdot 0$, it belongs to cell $k$ (using floor) [S2, unofficial].
*   **Coincident Crossings:** When the ray hits an edge or corner where multiple boundaries are crossed simultaneously, a fixed step priority (e.g., X, then Y, then Z) ensures deterministic traversal [S2, unofficial].
*   **Precision Errors:** Successive summation of $t_{max}$ can accumulate floating-point errors over long traversals [S5, unofficial]. Recalculating $t_{max}$ from scratch using total summation is an alternative but more expensive [S5, unofficial].
*   **Over-traversal:** In some implementations (e.g., LiDAR voxelization), traversal stops 1 voxel before the target point to prevent over-traversal [S1].

## Subtopics

### [DDA](2323-dda.md)
The Digital Differential Analyzer (DDA) is the standard grid traversal algorithm described in the main text. It operates directly on a flat uniform grid. [DDA](2323-dda.md)

### [Wide DDA Cell Traversal](5122-wide-dda-cell-traversal.md)
Wide DDA Cell Traversal is an algorithm for traversing a fixed 4x4x4 grid of "wide cells" within an Axis-Aligned Bounding Box (AABB) using a Digital Differential Analyzer (DDA) approach. It generates bounding boxes and t-intervals for each cell to optimize octree traversal by visiting only occupied child cells in ray order. It builds on [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) by providing the specific visitor callback mechanism and mask-based filtering required for wide octree nodes. [Wide DDA Cell Traversal](5122-wide-dda-cell-traversal.md)

### [Two-Step Discrete Ray Tracing](5204-two-step-discrete-ray-tracing.md)
Two-Step Discrete Ray Tracing is a ray traversal optimization for huge voxel spaces that replaces a single deep octree traversal with a two-level hierarchy of coarser and finer grids. It solves the performance penalty of traversing empty space at high resolution by first stepping through a coarse grid, then transitioning ray parameters into a finer sub-grid only when an occupied coarse voxel is hit. It differs from single-step discrete ray-tracing by using an octree to represent the scene at several resolutions, bypassing memory limits and optimizing traversal time. [Two-Step Discrete Ray Tracing](5204-two-step-discrete-ray-tracing.md)

### [Voxel Occupancy Mask](5242-voxel-occupancy-mask.md)
Voxel Occupancy Mask is an optimization for brick-level ray traversal that replaces per-brick occupancy flags with a 64-bit bitmask representing the occupancy of every voxel within a small brick. This allows the renderer to skip empty voxels inside an occupied brick without performing additional texture lookups for occupancy data. It differs from the parent [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) by introducing a finer-grained brick structure (4×4×4) and explicit bit-level occupancy checks. [Voxel Occupancy Mask](5242-voxel-occupancy-mask.md)

### [Zero-Crossing DDA Traversal](5474-zero-crossing-dda-traversal.md)
Zero-Crossing DDA Traversal performs empty-space ray-marching at all levels of a hierarchical grid using a Hierarchical Digital Differential Analyzer (HDDA) to detect the first voxel where a scalar field changes sign relative to the entry voxel's value. It solves the problem of efficiently skipping empty or uniform regions by combining coarse hierarchical traversal with narrow-band voxel-level detection. Unlike standard DDA methods that visit every voxel, this method returns immediately upon detecting a sign change, making it suitable for isosurface extraction or narrow-band volume rendering. It builds on [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) and differs from [DDA](2323-dda.md), [Two-Step Discrete Ray Tracing](5204-two-step-discrete-ray-tracing.md), [Voxel Occupancy Mask](5242-voxel-occupancy-mask.md), [Parallel 3D-DDA](6247-parallel-3d-dda.md), and [Nearest Neighbor Ray Traversal](8321-nearest-neighbor-ray-traversal.md) by halting traversal at the first sign change rather than sampling all active voxels. [Zero-Crossing DDA Traversal](5474-zero-crossing-dda-traversal.md)

### [Nearest Neighbor Ray Traversal](8321-nearest-neighbor-ray-traversal.md)
Nearest Neighbor Ray Traversal is a volume rendering method that samples data values at the center of each voxel immediately upon entry, requiring zero interpolation. It is distinguished from sibling methods like [Two-Step Discrete Ray Tracing](5204-two-step-discrete-ray-tracing.md) and [Zero-Crossing DDA Traversal](5474-zero-crossing-dda-traversal.md) by its lack of boundary-based sampling logic, prioritizing maximum speed over sampling uniformity. [Nearest Neighbor Ray Traversal](8321-nearest-neighbor-ray-traversal.md)

### [Uniform Grid Traversal](9652-uniform-grid-traversal.md)
Uniform Grid Traversal is a traversal method specifically optimized for uniform grids, leveraging the regularity of cell sizes to simplify boundary calculations and memory access patterns compared to non-uniform or hierarchical grids. [Uniform Grid Traversal](9652-uniform-grid-traversal.md)

### [Hardware/Implementation Optimization](10943-hardware-implementation-optimization.md)
Hardware/Implementation Optimization covers techniques to accelerate DDA traversal on GPUs and CPUs, such as using SIMD instructions for parallel axis comparisons, minimizing branching via branchless normal calculation, and optimizing memory coalescing during voxel access. [Hardware/Implementation Optimization](10943-hardware-implementation-optimization.md)

### Exact Grid Traversal
Exact Grid Traversal is an algorithm that captures all discrete grid points the line passes through. It is a naive implementation written by the author of the document [S4, unofficial].

### Surface Normal Calculation via Branchless Comparison
Surface Normal Calculation via Branchless Comparison is a method to determine the surface normal of a voxel hit by a ray by comparing the accumulated distances to the x and y planes, using branchless programming techniques [S5, unofficial]. It creates a comparison vector where each component is 1.0 if the ray hit that plane first (smaller `total_t`), and 0.0 otherwise, then calculates the normal components by multiplying the comparison vector components by the negative sign of the ray direction [S5, unofficial].

### SplineSplat
SplineSplat is a 3D ray-tracing algorithm that computes tomographic projections of a volume represented by a linear combination of shifted B-splines. It extends DDA to handle overlapping, compactly supported basis functions by using a neural network to compute basis contributions, avoiding the need for closed-form analytical expressions [S6].

### Direct Volume Rendering on Deformed Grids
Direct Volume Rendering on Deformed Grids traverses cells by finding the nearest intersection among the five remaining walls of the current cell after the first cell is identified [S7, unofficial]. It explicitly checks if the ray re-enters the grid after exiting, as this is possible in deformed grids [S7, unofficial]. A grid dimension limit of less than 255 elements per dimension is required to allow unique encoding of (i, j, k) indices into RGB values for the item buffer technique used to find the first hit [S7, unofficial]. If the viewpoint is sufficiently close, interior cell walls may be visible because outer walls are clipped by the hither plane, requiring all cell walls to be drawn rather than just the outer shell [S7, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels intersected by the ray. Each step involves constant-time arithmetic and comparisons [S5, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space per ray, as only a few state variables are maintained [S5, unofficial].
*   **Strengths:** Deterministic, no missed voxels, efficient for uniform grids, simple to implement [S1][S3, unofficial].
*   **Limitations:** Performance degrades if the ray traverses many empty voxels in sparse data (mitigated by hierarchical variants) [S1]. Floating-point precision errors can accumulate over long traversals [S5, unofficial].
*   **When to Use:** Uniform grids, dense volumes, or as a base for hierarchical traversal methods [S1][S3, unofficial].
*   **When Not to Use:** Sparse data where empty space skipping is critical (use hierarchical or occupancy-masked variants) [S1].

## Background
The 3D DDA algorithm was introduced by Amanatides and Woo in 1987 [S2, unofficial]. It is widely used in computer graphics for ray tracing, volume rendering, and voxelization [S1][S2, unofficial]. Variants are used in LiDAR processing [S1], CFD visualization [S7, unofficial], and tomographic reconstruction [S6].

## Sources
[S1] Ray Model and Voxelization | ai4ce/Occ4cast | DeepWiki, https://deepwiki.com/ai4ce/Occ4cast/3.2-ray-model-and-voxelization, fetched 2026-10-25
[S2] T03 — Fixture worlds, rays, and integer brushes, https://github.com/deranjer/spall/pull/4, fetched 2026-10-25 (unofficial)
[S3] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S4] Line Rasterization and Grid Traversal, https://trzy.org/line_rasterization/, fetched 2026-10-25 (unofficial)
[S5] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S6] SplineSplat: 3D Ray Tracing for Higher-Quality Tomography, https://ieeexplore.ieee.org/abstract/document/11516068/, fetched 2026-10-25
[S7] [PDF] Volume Rendering for Computational Fluid Dynamics: Initial Results, https://www.nas.nasa.gov/assets/nas/pdf/techreports/1991/rnr-91-026.pdf, fetched 2026-10-26 (unofficial)
