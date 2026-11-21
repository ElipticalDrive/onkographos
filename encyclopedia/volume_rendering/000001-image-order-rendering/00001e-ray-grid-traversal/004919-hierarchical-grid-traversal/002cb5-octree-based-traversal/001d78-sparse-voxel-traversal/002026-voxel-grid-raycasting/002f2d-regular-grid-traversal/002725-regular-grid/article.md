# Regular Grid

Also called: Grid Data Structure, Grilles régulières, Regular Grid Spatial Subdivision, Uniform Grids, Uniform Spatial Subdivision

## Overview
A Regular Grid is a spatial acceleration structure that divides a scene bounding box into a fixed-size 3D grid of cells (voxels) to reduce the number of ray-primitive intersection tests [S1, unofficial][S3][S4, unofficial]. It solves the problem of testing every object for every ray by only testing primitives stored in cells intersected by the ray [S4, unofficial]. Unlike hierarchical structures (e.g., BVH, Octree) that partition primitives into disjoint sets, the Regular Grid partitions space into disjoint sub-blocks, allowing primitives to belong to multiple sub-blocks [S5, unofficial]. It is distinguished by its constant-time access to cells and simple traversal logic, making it optimal for scenes with uniform data distribution on GPU [S7, unofficial]. Its linear build time makes it attractive for dynamic scenes requiring frequent rebuilds [S3].

## How it works
The method consists of two phases: Grid Construction (building the structure) and Ray Traversal (using the structure) [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** Scene primitives (with bounding boxes), scene bounding box $[p_1, p_2]$, grid resolution $(N_x, N_y, N_z)$ [S1, unofficial][S3].
*   **Outputs:** A 1D array representing the grid cells, where each cell contains a list of pointers to overlapping primitives [S1, unofficial].

### Data Structures
The grid is stored as a 1D array `m_Grid` of size $N_x \times N_y \times N_z$ [S1, unofficial]. Each element is a pointer to an `ObjectList` [S1, unofficial].
*   **Index Calculation:** For cell coordinates $(x, y, z)$, the index is:
    $$idx = x + y \cdot N_x + z \cdot N_x \cdot N_y$$
*   **ObjectList:** A linked list node containing:
    *   `primitive`: Pointer to the primitive object.
    *   `next`: Pointer to the next `ObjectList` in the chain [S1, unofficial].

### Grid Construction
1.  **Initialize:** Allocate `m_Grid` of size $N_x N_y N_z$ and set all entries to `NULL` [S1, unofficial].
2.  **Calculate Dimensions:** Compute voxel dimensions and their reciprocals for fast division:
    $$dx = \frac{p_2.x - p_1.x}{N_x}, \quad dx_{reci} = \frac{1}{dx}$$
    (Similarly for $dy, dz$ and $dy_{reci}, dz_{reci}$) [S1, unofficial].
3.  **Assign Primitives:** For each primitive $P$:
    *   Get its Axis-Aligned Bounding Box (AABB) $[b_{min}, b_{max}]$ [S1, unofficial].
    *   Calculate the range of grid cells overlapping this AABB [S1, unofficial]. Clamp coordinates to grid bounds $[0, N-1]$ to handle primitives outside or on the boundary [S1, unofficial]:
        $$x_1 = \max(0, \lfloor (b_{min}.x - p_1.x) \cdot dx_{reci} \rfloor)$$
        $$x_2 = \min(N_x - 1, \lfloor (b_{max}.x - p_1.x) \cdot dx_{reci} \rfloor + 1)$$
        (Similarly for $y_1, y_2$ and $z_1, z_2$) [S1, unofficial].
    *   Loop over all cells $(x, y, z)$ in the range $[x_1, x_2) \times [y_1, y_2) \times [z_1, z_2)$:
        *   Construct the AABB for the current cell [S1, unofficial].
        *   If the primitive intersects the cell AABB, add the primitive to the cell's `ObjectList` [S1, unofficial].
    *   *Note:* This mapping must be conservative; it may include more primitives than strictly necessary but never fewer [S1, S5, unofficial].

### Ray Traversal (3D-DDA)
The traversal identifies the sequence of cells intersected by a ray $r(t) = o + t \cdot d$ [S6, unofficial].

1.  **Ray-Grid Intersection:** Intersect the ray with the grid's bounding box $[p_1, p_2]$ to find entry $t_{enter}$ and exit $t_{exit}$ parameters [S1, unofficial].
    *   If $t_{exit} < 0$ or $t_{enter} > t_{exit}$, the ray misses the grid [S1, unofficial].
    *   If $t_{enter} < 0$, set $t_{enter} = 0$ (ray starts inside) [G].
2.  **Initialization:**
    *   Determine the starting point on the ray: $start\_point = o + d \cdot t_{enter}$ [S1, unofficial].
    *   Calculate the starting cell $(X, Y, Z)$:
        $$X = \lfloor \frac{start\_point.x - p_1.x}{dx} \rfloor$$
        (Similarly for $Y, Z$) [S1, unofficial].
    *   Calculate step directions:
        $$stepX = \text{sign}(d.x) \quad (\text{1 if } d.x \ge 0, -1 \text{ otherwise})$$
        (Similarly for $stepY, stepZ$) [S1, unofficial].
    *   Calculate delta distances (distance to cross one cell):
        $$tDeltaX = \frac{dx}{|d.x|} \text{ if } d.x \neq 0 \text{ else } \infty$$
        (Similarly for $tDeltaY, tDeltaZ$) [S1, unofficial].
    *   Calculate initial boundary distances ($tMax$):
        *   If $d.x > 0$:
            $$tMaxX = \frac{(X + 1) \cdot dx + p_1.x - start\_point.x}{d.x}$$
        *   If $d.x < 0$:
            $$tMaxX = \frac{X \cdot dx + p_1.x - start\_point.x}{d.x}$$
        *   If $d.x = 0$: $tMaxX = \infty$ [S1, unofficial].
        (Similarly for $tMaxY, tMaxZ$) [S1, unofficial].
3.  **Loop:**
    *   While $t_{enter} < t_{exit}$:
        *   Process the current cell $(X, Y, Z)$:
            *   If `m_Grid[idx]` is not `NULL`, iterate through the `ObjectList` [S1, unofficial].
            *   For each primitive, perform intersection test [S1, unofficial].
            *   *Deduplication:* To avoid testing the same primitive multiple times (if it spans multiple cells), use a "Mailbox" system: assign each ray a unique ID, and store the ray ID in the primitive's intersection cache [S6, unofficial]. Skip tests if the cache already contains the current ray ID.
            *   *Correctness Check:* If an intersection is found but the point is outside the current cell, continue advancing [S1, unofficial].
        *   Advance to the nearest boundary:
            *   Find axis $A \in \{X, Y, Z\}$ with minimum $tMaxA$ [S1, unofficial].
            *   Update $tMaxA \leftarrow tMaxA + tDeltaA$ [S1, unofficial].
            *   Update coordinate $A \leftarrow A + stepA$ [S1, unofficial].
            *   If the new coordinate is out of bounds, terminate [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class ObjectList:
    def __init__(self, primitive, next_node):
        self.primitive = primitive
        self.next = next_node

def intersect_aabb(origin, direction, min_pt, max_pt):
    # Standard slab method for ray-AABB intersection [G]
    t_min = (min_pt.x - origin.x) / direction.x if direction.x != 0 else float('-inf')
    t_max = (max_pt.x - origin.x) / direction.x if direction.x != 0 else float('inf')
    if t_min > t_max: t_min, t_max = t_max, t_min
    ty_min = (min_pt.y - origin.y) / direction.y if direction.y != 0 else float('-inf')
    ty_max = (max_pt.y - origin.y) / direction.y if direction.y != 0 else float('inf')
    if ty_min > ty_max: ty_min, ty_max = ty_max, ty_min
    if t_min > ty_max or ty_min > t_max: return float('inf'), float('-inf')
    t_min = max(t_min, ty_min)
    t_max = min(t_max, ty_max)
    tz_min = (min_pt.z - origin.z) / direction.z if direction.z != 0 else float('-inf')
    tz_max = (max_pt.z - origin.z) / direction.z if direction.z != 0 else float('inf')
    if tz_min > tz_max: tz_min, tz_max = tz_max, tz_min
    if t_min > tz_max or tz_min > t_max: return float('inf'), float('-inf')
    return max(t_min, 0.0), t_max

def test_primitive(prim, origin, direction, ray_id):
    # Standard ray-primitive intersection test [G]
    # Returns hit distance if intersected, else None
    return None

def build_regular_grid(primitives, p1, p2, Nx, Ny, Nz):
    # 1. Initialize grid
    grid_size = Nx * Ny * Nz
    grid = [None] * grid_size
    
    # 2. Calculate dimensions
    dx = (p2.x - p1.x) / Nx
    dy = (p2.y - p1.y) / Ny
    dz = (p2.z - p1.z) / Nz
    dx_rec = 1.0 / dx
    dy_rec = 1.0 / dy
    dz_rec = 1.0 / dz
    
    # 3. Assign primitives
    for prim in primitives:
        b_min, b_max = prim.get_aabb()
        
        # Calculate cell range
        x1 = max(0, math.floor((b_min.x - p1.x) * dx_rec))
        x2 = min(Nx - 1, math.floor((b_max.x - p1.x) * dx_rec) + 1)
        y1 = max(0, math.floor((b_min.y - p1.y) * dy_rec))
        y2 = min(Ny - 1, math.floor((b_max.y - p1.y) * dy_rec) + 1)
        z1 = max(0, math.floor((b_min.z - p1.z) * dz_rec))
        z2 = min(Nz - 1, math.floor((b_max.z - p1.z) * dz_rec) + 1)
        
        # Loop over candidate cells
        for x in range(x1, x2):
            for y in range(y1, y2):
                for z in range(z1, z2):
                    idx = x + y * Nx + z * Nx * Ny
                    
                    # Check intersection with cell AABB
                    cell_min = (p1.x + x * dx, p1.y + y * dy, p1.z + z * dz)
                    cell_max = (p1.x + (x + 1) * dx, p1.y + (y + 1) * dy, p1.z + (z + 1) * dz)
                    
                    if prim.intersects_aabb(cell_min, cell_max):
                        # Add to list (prepend)
                        new_node = ObjectList(prim, grid[idx])
                        grid[idx] = new_node
                        
    return grid

def traverse_regular_grid(ray_origin, ray_dir, grid, p1, p2, Nx, Ny, Nz, ray_id):
    # 1. Ray-Grid Intersection
    t_enter, t_exit = intersect_aabb(ray_origin, ray_dir, p1, p2)
    if t_exit < 0 or t_enter > t_exit:
        return None
    
    # Clamp t_enter to 0 if ray starts inside
    t_enter = max(t_enter, 0.0)
    
    # 2. Initialization
    start_point = ray_origin + ray_dir * t_enter
    
    X = math.floor((start_point.x - p1.x) / ((p2.x - p1.x) / Nx))
    Y = math.floor((start_point.y - p1.y) / ((p2.y - p1.y) / Ny))
    Z = math.floor((start_point.z - p1.z) / ((p2.z - p1.z) / Nz))
    
    # Step directions
    step_x = 1 if ray_dir.x >= 0 else -1
    step_y = 1 if ray_dir.y >= 0 else -1
    step_z = 1 if ray_dir.z >= 0 else -1
    
    # Delta distances
    dx = (p2.x - p1.x) / Nx
    dy = (p2.y - p1.y) / Ny
    dz = (p2.z - p1.z) / Nz
    
    t_delta_x = dx / abs(ray_dir.x) if ray_dir.x != 0 else float('inf')
    t_delta_y = dy / abs(ray_dir.y) if ray_dir.y != 0 else float('inf')
    t_delta_z = dz / abs(ray_dir.z) if ray_dir.z != 0 else float('inf')
    
    # Initial tMax
    if ray_dir.x > 0:
        t_max_x = ((X + 1) * dx + p1.x - start_point.x) / ray_dir.x
    elif ray_dir.x < 0:
        t_max_x = (X * dx + p1.x - start_point.x) / ray_dir.x
    else:
        t_max_x = float('inf')
        
    if ray_dir.y > 0:
        t_max_y = ((Y + 1) * dy + p1.y - start_point.y) / ray_dir.y
    elif ray_dir.y < 0:
        t_max_y = (Y * dy + p1.y - start_point.y) / ray_dir.y
    else:
        t_max_y = float('inf')
        
    if ray_dir.z > 0:
        t_max_z = ((Z + 1) * dz + p1.z - start_point.z) / ray_dir.z
    elif ray_dir.z < 0:
        t_max_z = (Z * dz + p1.z - start_point.z) / ray_dir.z
    else:
        t_max_z = float('inf')
        
    # 3. Loop
    while True:
        # Check bounds
        if not (0 <= X < Nx and 0 <= Y < Ny and 0 <= Z < Nz):
            break
            
        idx = X + Y * Nx + Z * Nx * Ny
        node = grid[idx]
        
        # Process cell
        while node is not None:
            prim = node.primitive
            # Mailbox check: skip if already tested for this ray
            if not prim.is_tested_for(ray_id):
                hit = test_primitive(prim, ray_origin, ray_dir, ray_id)
                if hit is not None:
                    return hit
            node = node.next
            
        # Advance to nearest boundary
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
                
    return None
```

### Parameters
*   **GRIDSIZE ($N$):** The number of cells along each axis. Total cells $N^3$ [S1, unofficial].
*   **Voxel Density ($D$):** Ratio of voxels to objects. $D=1.0$ is often inefficient; $D=20.0$ ("Uniform Grid Customary") is recommended for better performance in SPD scenes [S2, unofficial].
*   **Resolution:** Should strive to create cubical voxels by setting $N_x, N_y, N_z$ proportional to the scene bounding box dimensions [S2, unofficial][S3].

### Edge Cases and Pitfalls
*   **Intersection Outside Cell:** A primitive in cell $F$ may be intersected while the ray is in cell $F$, but the intersection point lies in cell $G$ [S1, unofficial]. Testing only the current cell misses this or requires redundant checks [S1, unofficial]. Solution: Continue advancing to subsequent cells until the intersection point is within the current cell or a closer intersection is found [S1, unofficial].
*   **Redundant Tests:** Primitives spanning multiple cells are tested multiple times [S6, unofficial]. Solution: Use "Mailboxes" (ray ID caching) to skip already-tested primitives [S6, unofficial].
*   **Sparse Scenes:** High memory usage due to empty cells [S5, unofficial]. Not suitable for very sparse scenes [S5, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the grid array plus storage for primitives in lists [S5, unofficial].

## Subtopics

### [Regular-Grid Mesh Intersection](15512-regular-grid-mesh-intersection.md)
Computes intersections between a ray and a regular-grid mesh with $O(P)$ complexity, exploiting structured topology for neural field rendering.

## Operating characteristics
*   **Time Complexity:**
    *   **Build:** Linear with respect to the number of primitives or grid size ($O(N_{prim} + N_{cells})$) [S3]. Allows interactive rebuilds for dynamic scenes [S3].
    *   **Traversal:** Depends on the number of cells traversed and primitives per cell. Setup cost $T_{setup}$ is significant; step cost $T_{step}$ degrades performance in large grids [S1, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the grid array plus storage for primitives in lists. High for sparse scenes as empty voxels consume memory [S5, unofficial].
*   **Performance:**
    *   On Intel Pentium II 350 MHz, 'tree' scene (513x513, depth 4): Uniform Grid $D=1.0$ took 1517.0s traversal; $D=20.0$ took 781.3s [S2, unofficial].
    *   Outperforms hierarchical grids for some SPD scenes with high voxel density [S2, unofficial].
    *   Optimal on GPU for uniform data distribution due to cached texture accesses (100x faster) [S7, unofficial].
*   **Strengths:** Simple implementation, fast build time, good for dynamic scenes, constant-time access [S3][S5, unofficial].
*   **Limitations:** High memory for sparse scenes, redundant tests for large primitives, not adaptive [S5, unofficial].

## Background
*   **Origin:** Introduced by Gla84 [S3].
*   **History:** Widely used in software ray tracers and as a starting point for graphics courses [S4, unofficial]. Referenced in Nvidia OptiX contexts as a foundational concept [S4, unofficial].
*   **Usage:** Used in particle-based simulations, computational fluid dynamics, and volume rendering [S3].

## Sources
[S1] flipcode - Raytracing Topics & Techniques, https://flipcode.com/archives/Raytracing_Topics_Techniques-Part_4_Spatial_Subdivisions.shtml, fetched 2026-10-24 (unofficial)
[S2] RTNv12n1.txt, https://realtimerendering.com/resources/RTNews/text/RTNv12n1.txt, fetched 2026-10-24 (unofficial)
[S3] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S4] Online Computer Graphics II: Rendering: Raytracing Acceleration: Acceleration Structures and Grids, https://www.youtube.com/watch?v=c01yvv4WVGs, fetched 2026-10-25 (unofficial)
[S5] 光线追踪小记:空间加速结构Regular Grid与3DDDA, https://zhuanlan.zhihu.com/p/415869768, fetched 2026-10-25 (unofficial)
[S6] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
[S7] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
