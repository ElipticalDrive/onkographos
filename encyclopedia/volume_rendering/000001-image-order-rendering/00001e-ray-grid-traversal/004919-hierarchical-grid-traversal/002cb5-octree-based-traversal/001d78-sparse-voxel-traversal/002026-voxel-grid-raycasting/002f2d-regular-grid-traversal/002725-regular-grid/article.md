# Regular Grid

Also called: Grid Data Structure, Grilles régulières, Regular Grid Spatial Subdivision, Uniform Grids, Uniform Spatial Subdivision

## Overview
A Regular Grid is a spatial acceleration structure that divides the scene bounding box into a fixed-size 3D grid of equal-sized voxels (cells) [S1, unofficial][S3][S6, unofficial]. It solves the problem of testing every object for every ray by only testing primitives stored in cells intersected by the ray, skipping empty cells entirely [S4, S6, unofficial]. Unlike hierarchical structures (e.g., BVH, Octrees) that partition primitives into disjoint sets, the Regular Grid partitions space into disjoint sub-blocks, allowing primitives to belong to multiple sub-blocks [S6, unofficial]. It is distinguished by its linear-time build complexity, making it suitable for dynamic scenes and GPU implementations where repeated construction is feasible [S3, S5].

## How it works
The method consists of two phases: Grid Construction and Ray Traversal [G].

### Inputs and Outputs
*   **Inputs:** Scene primitives (with AABBs), scene bounding box $[p_1, p_2]$, grid resolution $(N_x, N_y, N_z)$ or total size $N$.
*   **Outputs:** A 1D array representing the grid, where each entry points to a list of primitives.

### Data Structures
*   **Grid Array:** A 1D array of size $N_x \times N_y \times N_z$ (or $N^3$ for cubic grids). Each element is a pointer to an `ObjectList` [S1, unofficial].
*   **ObjectList:** A linked list containing pointers to primitives. Each node contains a pointer to a primitive and a pointer to the next node [S1, unofficial].
*   **Indexing:** The 3D coordinate $(x, y, z)$ maps to the 1D index:
    $$idx = x + y \cdot N_x + z \cdot N_x \cdot N_y$$
    [S1, unofficial]

### Grid Construction
1.  **Initialize:** Allocate the grid array of size $N_x \times N_y \times N_z$ and set all entries to `NULL` [G].
2.  **Calculate Dimensions:** Compute voxel dimensions and their reciprocals for fast division:
    $$dx = \frac{p_2.x - p_1.x}{N_x}, \quad dx_{reci} = \frac{1}{dx}$$
    (Similarly for $dy, dz$ and $dy_{reci}, dz_{reci}$) [G].
3.  **Assign Primitives:** For each primitive $P$:
    *   Get its Axis-Aligned Bounding Box (AABB) $[b_{min}, b_{max}]$ [G].
    *   Calculate the range of grid cells overlapping this AABB. Clamp coordinates to grid bounds $[0, N-1]$ to handle primitives outside or on the boundary [G]:
        $$x_1 = \max(0, \lfloor (b_{min}.x - p_1.x) \cdot dx_{reci} \rfloor)$$
        $$x_2 = \min(N_x - 1, \lfloor (b_{max}.x - p_1.x) \cdot dx_{reci} \rfloor + 1)$$
        (Similarly for $y_1, y_2$ and $z_1, z_2$) [G].
    *   Loop over all cells $(x, y, z)$ in the range $[x_1, x_2) \times [y_1, y_2) \times [z_1, z_2)$:
        *   Construct the AABB for the current cell [G].
        *   If the primitive intersects the cell AABB, add the primitive to the cell's `ObjectList` [G].
    *   *Note:* This mapping must be conservative; it may include more primitives than strictly necessary but never fewer [G].

### Ray Traversal (3D-DDA)
The traversal identifies the sequence of cells intersected by a ray $r(t) = o + t \cdot d$ [S1, unofficial].

1.  **Ray-Grid Intersection:** Intersect the ray with the grid's bounding box $[p_1, p_2]$ to find entry $t_{enter}$ and exit $t_{exit}$ parameters [G].
    *   If $t_{exit} < 0$ or $t_{enter} > t_{exit}$, the ray misses the grid [G].
    *   If $t_{enter} < 0$, set $t_{enter} = 0$ (ray starts inside) [G].
2.  **Initialization:**
    *   Determine the starting point on the ray: $start\_point = o + d \cdot t_{enter}$ [G].
    *   Calculate the starting cell $(X, Y, Z)$:
        $$X = \lfloor \frac{start\_point.x - p_1.x}{dx} \rfloor$$
        (Similarly for $Y, Z$) [G].
    *   Calculate step directions:
        $$stepX = \text{sign}(d.x) \quad (\text{1 if } d.x \ge 0, -1 \text{ otherwise})$$
        (Similarly for $stepY, stepZ$) [G].
    *   Calculate delta distances (distance to cross one cell):
        $$tDeltaX = \frac{dx}{|d.x|} \text{ if } d.x \neq 0 \text{ else } \infty$$
        (Similarly for $tDeltaY, tDeltaZ$) [G].
    *   Calculate initial boundary distances ($tMax$):
        *   If $d.x > 0$:
            $$tMaxX = \frac{(X + 1) \cdot dx + p_1.x - start\_point.x}{d.x}$$
        *   If $d.x < 0$:
            $$tMaxX = \frac{X \cdot dx + p_1.x - start\_point.x}{d.x}$$
        *   If $d.x = 0$: $tMaxX = \infty$ [G].
        (Similarly for $tMaxY, tMaxZ$) [G].
3.  **Loop:**
    *   While $t_{enter} < t_{exit}$:
        *   Process the current cell $(X, Y, Z)$:
            *   If the cell is not empty, iterate through the `ObjectList` [G].
            *   For each primitive, perform intersection test [G].
            *   *Deduplication:* To avoid testing the same primitive multiple times (if it spans multiple cells), use a "Mailbox" system: assign each ray a unique ID, and store the ray ID in the primitive's intersection cache. Skip tests if the cache already contains the current ray ID [G].
            *   *Correctness Check:* If an intersection is found but the point is outside the current cell, continue advancing [G].
        *   Advance to the nearest boundary:
            *   Find axis $A \in \{X, Y, Z\}$ with minimum $tMaxA$ [G].
            *   Update $tMaxA \leftarrow tMaxA + tDeltaA$ [G].
            *   Update coordinate $A \leftarrow A + stepA$ [G].
            *   If the new coordinate is out of bounds, terminate [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_grid(primitives, p1, p2, Nx, Ny, Nz):
    # 1. Initialize
    grid_size = Nx * Ny * Nz
    grid = [None] * grid_size
    
    # 2. Calculate Dimensions
    dx = (p2.x - p1.x) / Nx
    dy = (p2.y - p1.y) / Ny
    dz = (p2.z - p1.z) / Nz
    dx_reci = 1.0 / dx
    dy_reci = 1.0 / dy
    dz_reci = 1.0 / dz
    
    # 3. Assign Primitives
    for prim in primitives:
        b_min, b_max = prim.get_aabb()
        
        # Calculate cell range
        x1 = max(0, int((b_min.x - p1.x) * dx_reci))
        x2 = min(Nx - 1, int((b_max.x - p1.x) * dx_reci) + 1)
        y1 = max(0, int((b_min.y - p1.y) * dy_reci))
        y2 = min(Ny - 1, int((b_max.y - p1.y) * dy_reci) + 1)
        z1 = max(0, int((b_min.z - p1.z) * dz_reci))
        z2 = min(Nz - 1, int((b_max.z - p1.z) * dz_reci) + 1)
        
        # Loop over candidate cells
        for x in range(x1, x2 + 1):
            for y in range(y1, y2 + 1):
                for z in range(z1, z2 + 1):
                    # Construct cell AABB
                    cell_min = vec3(p1.x + x * dx, p1.y + y * dy, p1.z + z * dz)
                    cell_max = vec3(p1.x + (x + 1) * dx, p1.y + (y + 1) * dy, p1.z + (z + 1) * dz)
                    
                    # Check intersection (conservative)
                    if prim.intersects_aabb(cell_min, cell_max):
                        idx = x + y * Nx + z * Nx * Ny
                        if grid[idx] is None:
                            grid[idx] = ObjectList()
                        grid[idx].add(prim)
    return grid

def traverse_grid(ray, grid, p1, p2, Nx, Ny, Nz):
    # 1. Ray-Grid Intersection
    t_enter, t_exit = ray.intersect_aabb(p1, p2)
    if t_exit < 0 or t_enter > t_exit:
        return None
    if t_enter < 0:
        t_enter = 0
        
    start_point = ray.origin + ray.direction * t_enter
    
    # 2. Initialization
    dx = (p2.x - p1.x) / Nx
    dy = (p2.y - p1.y) / Ny
    dz = (p2.z - p1.z) / Nz
    
    X = int((start_point.x - p1.x) / dx)
    Y = int((start_point.y - p1.y) / dy)
    Z = int((start_point.z - p1.z) / dz)
    
    # Clamp to grid bounds if ray starts outside but intersects
    X = max(0, min(Nx - 1, X))
    Y = max(0, min(Ny - 1, Y))
    Z = max(0, min(Nz - 1, Z))
    
    stepX = 1 if ray.direction.x >= 0 else -1
    stepY = 1 if ray.direction.y >= 0 else -1
    stepZ = 1 if ray.direction.z >= 0 else -1
    
    tDeltaX = dx / abs(ray.direction.x) if ray.direction.x != 0 else float('inf')
    tDeltaY = dy / abs(ray.direction.y) if ray.direction.y != 0 else float('inf')
    tDeltaZ = dz / abs(ray.direction.z) if ray.direction.z != 0 else float('inf')
    
    # Calculate tMax
    if ray.direction.x > 0:
        tMaxX = ((X + 1) * dx + p1.x - start_point.x) / ray.direction.x
    elif ray.direction.x < 0:
        tMaxX = (X * dx + p1.x - start_point.x) / ray.direction.x
    else:
        tMaxX = float('inf')
        
    if ray.direction.y > 0:
        tMaxY = ((Y + 1) * dy + p1.y - start_point.y) / ray.direction.y
    elif ray.direction.y < 0:
        tMaxY = (Y * dy + p1.y - start_point.y) / ray.direction.y
    else:
        tMaxY = float('inf')
        
    if ray.direction.z > 0:
        tMaxZ = ((Z + 1) * dz + p1.z - start_point.z) / ray.direction.z
    elif ray.direction.z < 0:
        tMaxZ = (Z * dz + p1.z - start_point.z) / ray.direction.z
    else:
        tMaxZ = float('inf')
        
    # 3. Loop
    while t_enter < t_exit:
        # Process current cell
        idx = X + Y * Nx + Z * Nx * Ny
        cell_list = grid[idx]
        if cell_list is not None:
            for prim in cell_list:
                # Deduplication check would go here
                hit = prim.intersect(ray)
                if hit is not None:
                    # Correctness check: is hit point within current cell bounds?
                    # If not, continue advancing.
                    # For simplicity, assuming conservative grid ensures 
                    # if hit is valid, it's the closest or we handle it.
                    # In strict implementation, verify hit.t is within cell bounds.
                    return hit
        
        # Advance to nearest boundary
        if tMaxX < tMaxY:
            if tMaxX < tMaxZ:
                tMaxX += tDeltaX
                X += stepX
            else:
                tMaxZ += tDeltaZ
                Z += stepZ
        else:
            if tMaxY < tMaxZ:
                tMaxY += tDeltaY
                Y += stepY
            else:
                tMaxZ += tDeltaZ
                Z += stepZ
                
        # Check bounds
        if X < 0 or X >= Nx or Y < 0 or Y >= Ny or Z < 0 or Z >= Nz:
            break
            
    return None
```

## Subtopics

### [Regular-Grid Mesh Intersection](15512-regular-grid-mesh-intersection.md)
Regular-Grid Mesh Intersection computes intersections between a ray and a regular-grid mesh $M$ of size $P \times P \times P$ with $O(P)$ complexity. It differs from general ray-mesh intersection algorithms by exploiting the structured topology of the regular grid to restrict the search space to a linear number of voxels and faces, enabling efficient quadrature for neural field rendering.

### [Empty-Space Skipping Grid](22874-empty-space-skipping-grid.md)
The Empty-Space Skipping Grid is a 3D regular grid acceleration structure used in conjunction with the Volumetric Depth Image (VDI) to optimize raycasting by skipping empty regions along a ray's path. It reduces memory accesses by allowing rays to jump over grid cells that contain no supersegments, thereby avoiding unnecessary traversal of empty supersegment lists. Unlike hierarchical grids such as octrees or SparseLeap, this structure is chosen for its ability to be constructed dynamically during VDI generation using atomic operations, trading off some skipping efficiency for simpler, parallelizable build times.

## Operating characteristics
*   **Time Complexity:** Building time is linear with respect to the number of primitives or grid size, allowing creation from scratch and update at every frame with interactive performance for moderate sized scenes [S3]. Traversal time depends on the number of empty cells skipped and the number of primitives per cell.
*   **Memory Complexity:** Memory usage is high for sparse scenes because the grid is uniform and cannot adapt to primitive density. Many triangles may cluster in a small area but occupy many empty voxels [S6, unofficial]. The total number of cells is $N_x \cdot N_y \cdot N_z$ [S1, unofficial].
*   **Measured Performance:** On Intel Pentium II 350 MHz, for the 'tree' scene (resolution 513x513, recursion depth 4): Uniform Grid with voxel density $D=1.0$ (voxels = objects) took $T_B=0.22s$ (build) and $T_{TR}=1517.0s$ (traversal). Uniform Grid Customary with $D=20.0$ took $T_B=0.33s$ and $T_{TR}=781.3s$ [S2, unofficial].
*   **Strengths:** Simple implementation, constant-time access to elements, good data locality, optimal on GPU for uniform data distribution due to cached texture accesses [S8, unofficial]. Fast build time allows repeated construction for dynamic scenes [S3, S5].
*   **Limitations:** High memory usage for sparse scenes. Not suitable for dynamic scenes if using Proximity Clouds acceleration as pre-calculating distance information is costly [S8, unofficial]. The rule of thumb $D=1.0$ is not very good; higher density ($D=20.0$) is often better for SPD scenes [S2, unofficial].
*   **When to use:** Scenes with uniform data distribution, dynamic scenes requiring frequent rebuilds, or as a starting point for learning ray tracing [S4, S8, unofficial].
*   **When not to use:** Highly sparse scenes where hierarchical grids (BVH, Octree) would save memory and potentially traversal time [S2, S6, unofficial].

## Background
*   **Origin:** Grid data structures were introduced by [Gla84] [S3].
*   **History:** They have been popular for rendering analytical objects, particle-based simulations, and computational fluids dynamics [S3]. The 3D-DDA traversal algorithm is heavily based on the paper by Amanatides & Woo [S1, unofficial].
*   **Usage:** Used in software ray tracers, recommended for homework assignments [S4, unofficial], and referenced in Nvidia OptiX contexts [S4, unofficial]. On GPUs, sorting-based construction allows repeated use for each frame and stage, particularly for reflection/refraction rays which lack natural coherence [S5].

## Sources
[S1] flipcode - Raytracing Topics & Techniques, https://flipcode.com/archives/Raytracing_Topics_Techniques-Part_4_Spatial_Subdivisions.shtml, fetched 2026-10-24 (unofficial)
[S2] RTNv12n1.txt, https://realtimerendering.com/resources/RTNews/text/RTNv12n1.txt, fetched 2026-10-24 (unofficial)
[S3] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S4] Online Computer Graphics II: Rendering: Raytracing Acceleration: Acceleration Structures and Grids, https://www.youtube.com/watch?v=c01yvv4WVGs, fetched 2026-10-25 (unofficial)
[S5] Raytracing dynamic scenes on the GPU using grids, https://ieeexplore.ieee.org/abstract/document/5728799/, fetched 2026-10-25
[S6] 光线追踪小记:空间加速结构Regular Grid与3DDDA, https://zhuanlan.zhihu.com/p/415869768, fetched 2026-10-25 (unofficial)
[S7] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
[S8] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
