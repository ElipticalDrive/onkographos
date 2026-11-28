# Regular Grid Traversal

Also called: Grid Traversal Algorithm, 3D-DDA, Digital Differential Analyzer (for grids)

## Overview
Regular Grid Traversal is a ray-scene intersection acceleration technique that partitions a scene bounding box into a fixed-size 3D grid of cells, returning the sequence of cells intersected by a ray to minimize primitive intersection tests [S1]. It solves the problem of testing every object for every ray by only testing primitives stored in intersected cells . Unlike hierarchical structures (e.g., BVH, Octree) that partition primitives into disjoint sets, the Regular Grid partitions space into disjoint sub-blocks, allowing primitives to belong to multiple sub-blocks . It is distinguished by its constant-time access to cells and simple traversal logic, making it optimal for scenes with uniform data distribution or dynamic scenes requiring frequent rebuilds .

## Shared principles
The method consists of two phases: Grid Construction (building the structure) and Ray Traversal (using the structure).

### Grid Construction
1.  **Initialize:** Allocate a 1D array `m_Grid` of size $N_x \times N_y \times N_z$ and set all entries to `NULL` [S1].
2.  **Calculate Dimensions:** Compute voxel dimensions and their reciprocals for fast division:
    $$dx = \frac{p_2.x - p_1.x}{N_x}, \quad dx_{reci} = \frac{1}{dx}$$
    (Similarly for $dy, dz$ and $dy_{reci}, dz_{reci}$) [S1].
3.  **Assign Primitives:** For each primitive $P$:
    *   Get its Axis-Aligned Bounding Box (AABB) $[b_{min}, b_{max}]$ [S1].
    *   Calculate the range of grid cells overlapping this AABB. Clamp coordinates to grid bounds $[0, N-1]$ to handle primitives outside or on the boundary [S1]:
        $$x_1 = \max(0, \lfloor (b_{min}.x - p_1.x) \cdot dx_{reci} \rfloor)$$
        $$x_2 = \min(N_x - 1, \lfloor (b_{max}.x - p_1.x) \cdot dx_{reci} \rfloor + 1)$$
        (Similarly for $y_1, y_2$ and $z_1, z_2$) [S1].
    *   Loop over all cells $(x, y, z)$ in the range $[x_1, x_2) \times [y_1, y_2) \times [z_1, z_2)$:
        *   Construct the AABB for the current cell [S1].
        *   If the primitive intersects the cell AABB, add the primitive to the cell's `ObjectList` [S1].
    *   *Note:* This mapping must be conservative; it may include more primitives than strictly necessary but never fewer [S1].

### Ray Traversal (3D-DDA)
The traversal identifies the sequence of cells intersected by a ray $r(t) = o + t \cdot d$.

1.  **Ray-Grid Intersection:** Intersect the ray with the grid's bounding box $[p_1, p_2]$ to find entry $t_{enter}$ and exit $t_{exit}$ parameters [S1].
    *   If $t_{exit} < 0$ or $t_{enter} > t_{exit}$, the ray misses the grid [S1].
    *   If $t_{enter} < 0$, set $t_{enter} = 0$ (ray starts inside) [G].
2.  **Initialization:**
    *   Determine the starting point on the ray: $start\_point = o + d \cdot t_{enter}$ [S1].
    *   Calculate the starting cell $(X, Y, Z)$:
        $$X = \lfloor \frac{start\_point.x - p_1.x}{dx} \rfloor$$
        (Similarly for $Y, Z$) [S1].
    *   Calculate step directions:
        $$stepX = \text{sign}(d.x) \quad (\text{1 if } d.x \ge 0, -1 \text{ otherwise})$$
        (Similarly for $stepY, stepZ$) [S1].
    *   Calculate delta distances (distance to cross one cell):
        $$tDeltaX = \frac{dx}{|d.x|} \text{ if } d.x \neq 0 \text{ else } \infty$$
        (Similarly for $tDeltaY, tDeltaZ$) [S1].
    *   Calculate initial boundary distances ($tMax$):
        *   If $d.x > 0$:
            $$tMaxX = \frac{(X + 1) \cdot dx + p_1.x - start\_point.x}{d.x}$$
        *   If $d.x < 0$:
            $$tMaxX = \frac{X \cdot dx + p_1.x - start\_point.x}{d.x}$$
        *   If $d.x = 0$: $tMaxX = \infty$ [S1].
        (Similarly for $tMaxY, tMaxZ$) [S1].
3.  **Loop:**
    *   While $t_{enter} < t_{exit}$:
        *   Process the current cell $(X, Y, Z)$:
            *   If `m_Grid[idx]` is not `NULL`, iterate through the `ObjectList`.
            *   For each primitive, perform intersection test.
            *   *Deduplication:* To avoid testing the same primitive multiple times (if it spans multiple cells), use a "Mailbox" system: assign each ray a unique ID, and store the ray ID in the primitive's intersection cache. Skip tests if the cache already contains the current ray ID .
            *   *Correctness Check:* If an intersection is found but the point is outside the current cell, continue advancing [S1].
        *   Advance to the nearest boundary:
            *   Find axis $A \in \{X, Y, Z\}$ with minimum $tMaxA$.
            *   Update $tMaxA \leftarrow tMaxA + tDeltaA$.
            *   Update coordinate $A \leftarrow A + stepA$.
            *   If the new coordinate is out of bounds, terminate [S1].

## Subtopics

### [Regular Grid](3950-regular-grid.md)
A Regular Grid is a spatial acceleration structure that divides a scene bounding box into a fixed-size 3D grid of cells (voxels) to reduce the number of ray-primitive intersection tests. It solves the problem of testing every object for every ray by only testing primitives stored in cells intersected by the ray. Unlike hierarchical structures (e.g., BVH, Octree) that partition primitives into disjoint sets, the Regular Grid partitions space into disjoint sub-blocks, allowing primitives to belong to multiple sub-blocks. It is distinguished by its constant-time access to cells and simple traversal logic, making it optimal for scenes with uniform data distribution or dynamic scenes requiring frequent rebuilds.

### [Voxel-based Ray Tracing for GI and Specular Reflections](4091-voxel-based-ray-tracing-for-gi-and-specular-reflections.md)
Voxel-based Ray Tracing is a global illumination (GI) and specular reflection fallback method that performs Digital Differential Analyzer (DDA) traversal through a sparse, block-occupancy voxel grid to sample indirect lighting and environment reflections. It serves as a drop-in replacement for Screen-Space Ray Tracing (SSRT) for GI and a fallback for Screen-Space Reflections (SSR) when screen-space coverage is incomplete, combining sky occlusion with Light Propagation Volumes (LPV) colour bleeding. Unlike standard voxel raycasting which often relies on volume density, this method operates on a binary block-occupancy grid to determine surface hits for lighting queries.

### [Optimised Voxel Ray Marching](4240-optimised-voxel-ray-marching.md)
Optimised Voxel Ray Marching is a ray traversal algorithm for voxel-based scenes that uses fixed-length steps along the ray to traverse a 3D grid of voxels, checking for occupancy at each step. It solves the problem of inefficiently stepping through empty space in large voxel worlds by optimizing memory access patterns, specifically by keeping block lookups within the current working chunk to minimize bitshifts, masks, and multiplication operations. It sets itself apart from [Regular Grid Traversal](5939-regular-grid-traversal.md) by using a chunked memory layout that reduces boundary calculation overhead, and from [Chunked Voxel Traversal](8311-chunked-voxel-traversal.md) by relying on fixed-size inner grids which simplifies implementation and memory layout.

## Background
The Regular Grid Traversal algorithm is a foundational technique in ray tracing, historically developed to determine the cells traversed by a ray in a grid [S1]. Algorithms for grid traversal are quite similar to rasterization algorithms (such as Bresenham line draw algorithm [Bre65]), but unlike Bresenham which produces visually acceptable lines in the shortest time without finding all pixels traversed, grid traversal must find all cells traversed by the ray segment [S1]. A naive 3D extension of Bresenham is not suitable as a grid traversal algorithm [S1].

Traversal time is a critical metric, often modeled by two constants: $T_{setup}$ (time to setup traversal algorithm) and $T_{step}$ (time spent to advance from a cell to its neighbor without checking intersection) [S1]. An algorithm with low $T_{setup}$ usually performs better for low resolution grids, whereas high $T_{step}$ provides bad performance for larger grids [S1].

Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
