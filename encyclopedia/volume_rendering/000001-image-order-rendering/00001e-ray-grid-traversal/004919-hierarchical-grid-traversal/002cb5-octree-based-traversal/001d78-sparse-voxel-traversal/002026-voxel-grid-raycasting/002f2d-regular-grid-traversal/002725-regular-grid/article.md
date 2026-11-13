# Regular Grid

Also called: Grilles régulières, Regular Grid Spatial Subdivision

## Overview
A Regular Grid is a spatial acceleration structure that divides a scene's Axis-Aligned Bounding Box (AABB) into a fixed-size, uniform 3D grid of cells (voxels) [S1, S2, unofficial]. It accelerates ray tracing by allowing rays to skip empty cells, reducing primitive intersection tests [S1, S2, unofficial]. Unlike BVH, which partitions primitives into disjoint sets, the Regular Grid partitions space into disjoint sub-blocks, allowing primitives to belong to multiple sub-blocks [S2, S3, unofficial]. It is optimal on GPUs for scenes with uniform data distribution due to high spatial locality and cache efficiency [S3, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs**: Scene primitives, scene bounding box defined by minimum corner $p_1$ and maximum corner $p_2$ [S1, unofficial], and grid resolution $N$ (GRIDSIZE), the number of cells along each axis [S1, unofficial].
*   **Outputs**: A 1D array `m_Grid` representing the voxel grid [S1, unofficial], and for a given ray, the closest intersection point with a primitive [S1, unofficial].

### Data Structures
**1. The Grid Array (`m_Grid`)**
*   **Layout**: A 1D array of pointers to `ObjectList` [S1, unofficial].
*   **Size**: $N^3$ elements [S1, unofficial].
*   **Indexing**: The 3D coordinate $(x, y, z)$ maps to the 1D index: $idx = x + y \cdot N + z \cdot N^2$ [S1, unofficial].
*   **Content**: Each element points to a linked list of primitives intersecting that voxel. If the voxel is empty, the pointer is `NULL` [S1, unofficial].

**2. Object List (`ObjectList`)**
*   **Fields**: `primitive` (pointer to the primitive) and `next` (pointer to the next `ObjectList` in the chain) [S1, unofficial].

**3. Ray State (during traversal)**
*   **Current Cell**: Integers $(X, Y, Z)$ [G].
*   **Distances**: $tMaxX, tMaxY, tMaxZ$ (distance along the ray to the next cell boundary in each axis) and $tDeltaX, tDeltaY, tDeltaZ$ (distance along the ray to cross one full cell in each axis) [G].
*   **Steps**: Integers $stepX, stepY, stepZ$ ($+1$ or $-1$) indicating direction [G].

### Grid Construction
The goal is to map each primitive to all voxels it intersects. This mapping must be conservative: it may include more primitives than strictly necessary but never fewer [S2, unofficial].
1.  **Initialize Grid**: Allocate `m_Grid` of size $N^3$ and set all pointers to `NULL` [S1, unofficial].
2.  **Define Bounds**: Set scene bounds $p_1$ (min) and $p_2$ (max) [S1, unofficial].
3.  **Calculate Cell Dimensions**:
    $$ dx = \frac{p_2.x - p_1.x}{N}, \quad dy = \frac{p_2.y - p_1.y}{N}, \quad dz = \frac{p_2.z - p_1.z}{N} $$
    Calculate reciprocals: $dx_{reci} = 1/dx$, $dy_{reci} = 1/dy$, $dz_{reci} = 1/dz$ [S1, unofficial].
4.  **Map Primitives**: For each primitive $p$:
    a. Get its AABB: $bound_{min}$ and $bound_{max}$ [S1, unofficial].
    b. Calculate grid cell coordinates $(x_1, y_1, z_1)$ containing $bound_{min}$ and $(x_2, y_2, z_2)$ containing $bound_{max}$:
       $$ x_1 = \lfloor (bound_{min}.x - p_1.x) \cdot dx_{reci} \rfloor, \quad x_2 = \lfloor (bound_{max}.x - p_1.x) \cdot dx_{reci} \rfloor + 1 $$
       (Repeat for $y$ and $z$) [S1, unofficial].
    c. **Clamp**: Ensure coordinates are within $[0, N-1]$: $x_1 = \max(0, \min(x_1, N-1))$, $x_2 = \max(0, \min(x_2, N-1))$ (Repeat for $y, z$) [S1, unofficial].
    d. **Insert**: Loop over all cells $(x, y, z)$ from $(x_1, y_1, z_1)$ to $(x_2, y_2, z_2)$ [S1, unofficial]. For each cell, add primitive $p$ to the cell's object list [S1, unofficial].

### Grid Traversal
Uses 3D-DDA to step through the grid [G].
1.  **Setup**:
    a. Determine the starting cell $(X, Y, Z)$ for the ray origin $O$: $X = \lfloor (O.x - p_1.x)/dx \rfloor$, etc. [G].
    b. If the ray origin is outside the grid, advance the ray to the first intersection with the grid's bounding box. If it misses, terminate [S1, unofficial].
    c. Calculate $tDelta$ values (constant per ray): $tDeltaX = dx / |dir_x|$, $tDeltaY = dy / |dir_y|$, $tDeltaZ = dz / |dir_z|$ [G].
    d. Calculate $tMax$ values (distance to first boundary):
       $$ tMaxX = \frac{(X + (dir_x > 0 ? 1 : 0)) \cdot dx + p_1.x - O.x}{dir_x} $$
       (Repeat for $Y, Z$) [G].
    e. Determine step directions: $stepX = 1$ if $dir_x > 0$ else $-1$ (Repeat for $Y, Z$) [G].
2.  **Traversal Loop**:
    While the ray is within the grid bounds:
    a. **Process Cell**: Let current cell be $(X, Y, Z)$ [S1, unofficial]. If `m_Grid[idx]` is not `NULL`, iterate through the `ObjectList` and perform intersection tests [S1, unofficial].
    b. **Pitfall Handling**: If an intersection is found but the point lies outside the current cell, advance the ray to subsequent cells and test their primitives until the intersection point is within the current cell or a closer intersection is found [S1, unofficial]. Mark primitives as tested to avoid redundant checks across multiple voxels [S2, unofficial].
    c. **Advance Ray**: Find $t_{min} = \min(tMaxX, tMaxY, tMaxZ)$ [S1, unofficial]. If $t_{min} > t_{far}$, break. Update coordinates and $tMax$: if $tMaxX == t_{min}$, $tMaxX += tDeltaX, X += stepX$; else if $tMaxY == t_{min}$, $tMaxY += tDeltaY, Y += stepY$; else $tMaxZ += tDeltaZ, Z += stepZ$ [S1, unofficial]. Check bounds: if $X, Y, Z$ are outside $[0, N-1]$, terminate [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_regular_grid(primitives, p1, p2, N):
    m_grid = [None] * (N * N * N)
    dx = (p2[0] - p1[0]) / N
    dy = (p2[1] - p1[1]) / N
    dz = (p2[2] - p1[2]) / N
    dx_reci = 1.0 / dx
    dy_reci = 1.0 / dy
    dz_reci = 1.0 / dz
    for prim in primitives:
        bound_min, bound_max = prim.get_aabb()
        x1 = int((bound_min[0] - p1[0]) * dx_reci)
        x2 = int((bound_max[0] - p1[0]) * dx_reci) + 1
        y1 = int((bound_min[1] - p1[1]) * dy_reci)
        y2 = int((bound_max[1] - p1[1]) * dy_reci) + 1
        z1 = int((bound_min[2] - p1[2]) * dz_reci)
        z2 = int((bound_max[2] - p1[2]) * dz_reci) + 1
        x1 = max(0, min(x1, N - 1))
        x2 = max(0, min(x2, N - 1))
        y1 = max(0, min(y1, N - 1))
        y2 = max(0, min(y2, N - 1))
        z1 = max(0, min(z1, N - 1))
        z2 = max(0, min(z2, N - 1))
        # Source code uses x < x2; range(x1, x2 + 1) ensures inclusive upper bound per notes
        for x in range(x1, x2 + 1):
            for y in range(y1, y2 + 1):
                for z in range(z1, z2 + 1):
                    idx = x + y * N + z * N * N
                    if m_grid[idx] is None:
                        m_grid[idx] = []
                    m_grid[idx].append(prim)
    return m_grid

def trace_ray_grid(ray_origin, ray_dir, p1, p2, N, m_grid):
    dx = (p2[0] - p1[0]) / N
    dy = (p2[1] - p1[1]) / N
    dz = (p2[2] - p1[2]) / N
    X = int((ray_origin[0] - p1[0]) / dx)
    Y = int((ray_origin[1] - p1[1]) / dy)
    Z = int((ray_origin[2] - p1[2]) / dz)
    t_delta_x = abs(dx / ray_dir[0]) if ray_dir[0] != 0 else float('inf')
    t_delta_y = abs(dy / ray_dir[1]) if ray_dir[1] != 0 else float('inf')
    t_delta_z = abs(dz / ray_dir[2]) if ray_dir[2] != 0 else float('inf')
    step_x = 1 if ray_dir[0] > 0 else -1
    step_y = 1 if ray_dir[1] > 0 else -1
    step_z = 1 if ray_dir[2] > 0 else -1
    t_max_x = ((X + (1 if ray_dir[0] > 0 else 0)) * dx + p1[0] - ray_origin[0]) / ray_dir[0] if ray_dir[0] != 0 else float('inf')
    t_max_y = ((Y + (1 if ray_dir[1] > 0 else 0)) * dy + p1[1] - ray_origin[1]) / ray_dir[1] if ray_dir[1] != 0 else float('inf')
    t_max_z = ((Z + (1 if ray_dir[2] > 0 else 0)) * dz + p1[2] - ray_origin[2]) / ray_dir[2] if ray_dir[2] != 0 else float('inf')
    closest_dist = float('inf')
    tested_primitives = set()
    while 0 <= X < N and 0 <= Y < N and 0 <= Z < N:
        idx = X + Y * N + Z * N * N
        if m_grid[idx] is not None:
            for prim in m_grid[idx]:
                if prim in tested_primitives:
                    continue
                t, point = prim.intersect(ray_origin, ray_dir)
                if t is not None and t < closest_dist:
                    cell_min_x = p1[0] + X * dx
                    cell_max_x = p1[0] + (X + 1) * dx
                    cell_min_y = p1[1] + Y * dy
                    cell_max_y = p1[1] + (Y + 1) * dy
                    cell_min_z = p1[2] + Z * dz
                    cell_max_z = p1[2] + (Z + 1) * dz
                    if (cell_min_x <= point[0] < cell_max_x and
                        cell_min_y <= point[1] < cell_max_y and
                        cell_min_z <= point[2] < cell_max_z):
                        closest_dist = t
                        tested_primitives.add(prim)
        if t_max_x < t_max_y:
            if t_max_x < t_max_z:
                t_max_x += t_delta_x
                X += step_x
            else:
                t_max_z += t_delta_z
                Z += step_z
        else:
            if t_max_y < t_max_z:
                t_max_y += t_delta_y
                Y += step_y
            else:
                t_max_z += t_delta_z
                Z += step_z
    return closest_dist
```

### Parameters
*   **GRIDSIZE ($N$)**: The number of cells along each axis. The total number of cells is $N^3$ [S1, unofficial].

### Edge Cases and Pitfalls
1.  **Intersection Outside Cell**: A primitive stored in cell $F$ may be intersected by a ray while traversing cell $F$, but the intersection point lies in cell $G$. Testing only the current cell misses this or requires redundant checks [S1, unofficial]. Solution: Verify if the intersection point lies within the current cell's bounds. If not, continue advancing to subsequent cells and test their primitives until the intersection point is within the current cell or a closer intersection is found [S1, unofficial].
2.  **Redundant Tests**: A single triangle may span multiple voxels [S2, unofficial]. Without marking, the same primitive may be tested multiple times [S2, unofficial]. Solution: Mark primitives as tested for the current ray or rely on the closest intersection logic combined with cell-boundary checks [S2, unofficial].
3.  **Ray Origin Outside Grid**: If the ray starts outside the grid, $tMax$ calculations must account for the entry point into the grid's bounding box. If the ray misses the grid entirely, traversal should terminate immediately [S1, unofficial].
4.  **Zero Direction Component**: If a ray direction component is zero (e.g., $dir_x = 0$), $tDelta$ and $tMax$ calculations for that axis must be handled to avoid division by zero. The ray never crosses a boundary in that axis [G].

## Operating characteristics
*   **Time Complexity**: Construction: $O(P \cdot V)$, where $P$ is primitives and $V$ is average voxels per primitive [S1, unofficial]. Naive construction is $O(N^3 \cdot P)$ [S1, unofficial]. Traversal: $O(C + K \cdot I)$, where $C$ is cells visited, $K$ is average primitives per cell, $I$ is intersection cost [G].
*   **Memory Complexity**: $O(N^3 + P \cdot V)$ [S2, unofficial]. The grid array is $N^3$ pointers. Memory usage is high for sparse scenes because the grid is uniform and cannot adapt to primitive density; many empty voxels are allocated [S2, unofficial].
*   **Performance**: GPU: Considered optimal on GPU for scenes with uniform data distribution due to high spatial locality. Successive elements are spatially close in memory, leading to a high probability of cached texture accesses (up to 100x faster than non-localized access) [S3, unofficial]. CPU: Can be slower than hierarchical structures for sparse or clustered scenes due to poor cache locality and high memory bandwidth usage for empty cells [S2, unofficial].
*   **Strengths**: Simple implementation, constant-time access to cells, good data locality on GPU, effective for secondary rays in uniform scenes [S1, S3, unofficial].
*   **Limitations**: High memory usage for sparse scenes, no adaptivity, performance degrades with high primitive density in small areas, not suitable for dynamic scenes if using advanced acceleration like Proximity Clouds (too costly to update) [S2, S3, unofficial].

## Background
*   **Origin**: The Regular Grid is one of the earliest spatial subdivision structures used in ray tracing. It was prominently featured in early GPU ray tracing experiments [S3, unofficial]. The traversal algorithm (3D-DDA) is heavily based on the paper by Amanatides & Woo [S1, unofficial].
*   **History**: Early implementations used brute-force cell-primitive intersection checks during construction, which was slow for large scenes [S1, unofficial]. Optimizations involved mapping primitives to voxels based on their AABBs first [S1, unofficial]. The structure was favored in early GPU ray tracers due to its regular memory access pattern, which aligned well with texture caches [S3, unofficial].
*   **Usage**: Used in early GPU ray tracers, volume rendering, and scenes with uniform object distribution. It serves as a baseline for more complex structures like Octrees and K-D trees [S2, S3, unofficial].

## Sources
[S1] flipcode - Raytracing Topics & Techniques, https://flipcode.com/archives/Raytracing_Topics_Techniques-Part_4_Spatial_Subdivisions.shtml, fetched 2026-10-24 (unofficial)
[S2] 光线追踪小记:空间加速结构Regular Grid与3DDDA, https://zhuanlan.zhihu.com/p/415869768, fetched 2026-10-25 (unofficial)
[S3] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
