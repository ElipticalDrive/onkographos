# Regular Grid

Also called: Grid Data Structure, Grilles régulières, Regular Grid Spatial Subdivision, Uniform Grids

## Overview
A Regular Grid is a spatial acceleration structure that partitions the scene's Axis-Aligned Bounding Box (AABB) into a fixed-size 3D grid of uniform cells (voxels) [S1, unofficial][S2][S3, unofficial]. It reduces ray-primitive intersection tests by storing pointers to primitives in the cells they intersect, allowing rays to skip empty cells during traversal [S1, S3, unofficial]. Unlike hierarchical structures or BVHs, it offers constant-time access to cells and excellent data locality, making it optimal for GPU execution on scenes with uniform data distribution, though it suffers from high memory overhead in sparse scenes [S2][S3, S5, unofficial].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   A set of scene primitives.
    *   The scene's bounding box defined by minimum corner $p_1$ and maximum corner $p_2$ [S1, S4, unofficial].
    *   Grid resolution parameters: $GRIDSIZE$ (if cubic) or $mx, my, mz$ (if non-cubic) [S1, unofficial][S2].
*   **Outputs:**
    *   A 1D array `m_Grid` of size $GRIDSIZE^3$ (or $mx \cdot my \cdot mz$), where each element is a pointer to a linked list of primitives [S1, unofficial][S2].

### Data Structures
The grid is stored as a flat 1D array to ensure spatial locality [S5, unofficial].
*   **`m_Grid`**: Array of pointers to `ObjectList`.
    *   Size: $N^3$ where $N = GRIDSIZE$ (cubic) or $mx \cdot my \cdot mz$ (non-cubic) [S1, unofficial][S2].
    *   Index calculation for cell $(x, y, z)$:
        $$idx = x + y \cdot GRIDSIZE + z \cdot GRIDSIZE \cdot GRIDSIZE$$
        [S1, unofficial]
*   **`ObjectList`**: Linked list node.
    *   Fields:
        *   `primitive`: Pointer to the primitive object.
        *   `next`: Pointer to the next `ObjectList` in the chain [S1, unofficial].

### Parameters
*   **$GRIDSIZE$**: Number of cells along each axis for a cubic grid. Total cells = $GRIDSIZE^3$ [S1, unofficial].
*   **$mx, my, mz$**: Number of cells along x, y, z axes respectively, used when the scene bounding box is not cubic but cubic voxels are desired [S2].
*   **Typical Values**: $m$ is determined by scene size and resolution requirements to balance build time and traversal time [S2]. Not given in the sources for specific numeric ranges.

### Algorithm Steps

#### 1. Grid Construction
1.  **Initialize**: Allocate `m_Grid` of size $GRIDSIZE^3$ and set all entries to `NULL` [S1, unofficial].
2.  **Define Bounds**: Determine scene bounding box $p_1$ (min) and $p_2$ (max) [S1, S4, unofficial].
3.  **Calculate Cell Dimensions**:
    $$dx = \frac{p_2.x - p_1.x}{GRIDSIZE}, \quad dx_{reci} = \frac{1}{dx}$$
    $$dy = \frac{p_2.y - p_1.y}{GRIDSIZE}, \quad dy_{reci} = \frac{1}{dy}$$
    $$dz = \frac{p_2.z - p_1.z}{GRIDSIZE}, \quad dz_{reci} = \frac{1}{dz}$$
    [S1, unofficial]
4.  **Map Primitives**: For each primitive $p$:
    a. Get its AABB bounds $bv_1$ (min) and $bv_2$ (max) [S1, unofficial].
    b. Calculate the range of grid cells intersected by the primitive's AABB:
       $$x_1 = \max(0, \lfloor (bv_1.x - p_1.x) \cdot dx_{reci} \rfloor)$$
       $$x_2 = \min(GRIDSIZE - 1, \lfloor (bv_2.x - p_1.x) \cdot dx_{reci} \rfloor + 1)$$
       (Repeat for $y$ and $z$ using $dy_{reci}$ and $dz_{reci}$) [S1, unofficial].
       *Note: This mapping must be conservative; it may include more primitives than strictly necessary but never fewer [S3, unofficial].*
    c. Iterate through all cells $(x, y, z)$ in the range $[x_1, x_2) \times [y_1, y_2) \times [z_1, z_2)$:
       i. Calculate index $idx = x + y \cdot GRIDSIZE + z \cdot GRIDSIZE^2$ [S1, unofficial].
       ii. Add primitive $p$ to the linked list at `m_Grid[idx]` [S1, unofficial].

#### 2. Ray Traversal Setup
For a ray $r(t) = o + t \cdot d$:
1.  **Determine Start Cell**:
    *   If the ray origin $o$ is inside the grid, calculate initial cell $(X, Y, Z)$:
        $$X = \lfloor \frac{o.x - p_1.x}{dx} \rfloor, \quad Y = \lfloor \frac{o.y - p_1.y}{dy} \rfloor, \quad Z = \lfloor \frac{o.z - p_1.z}{dz} \rfloor$$
    *   If the ray origin is outside, advance the ray to the first intersection with the grid's bounding box. If it misses the grid, terminate [S1, unofficial].
2.  **Calculate Step Directions**:
    $$stepX = \begin{cases} 1 & \text{if } d.x \ge 0 \\ -1 & \text{if } d.x < 0 \end{cases}$$
    (Repeat for $stepY$ and $stepZ$) [S1, unofficial].
3.  **Calculate Distances**:
    *   $tMaxX, tMaxY, tMaxZ$: Distance along the ray to the first cell boundary in each axis.
        $$tMaxX = \frac{(X + (stepX > 0 ? 1 : 0)) \cdot dx + p_1.x - o.x}{d.x}$$
        (Simplified logic: distance to the next grid plane in the direction of travel) [S1, S3, unofficial].
    *   $tDeltaX, tDeltaY, tDeltaZ$: Distance along the ray to cross one full cell in each axis.
        $$tDeltaX = \frac{dx}{|d.x|}, \quad tDeltaY = \frac{dy}{|d.y|}, \quad tDeltaZ = \frac{dz}{|d.z|}$$
        [S1, S3, unofficial]

#### 3. Grid Traversal Loop
1.  **Check Current Cell**:
    *   If `m_Grid[X, Y, Z]` is not `NULL`, iterate through the primitive list.
    *   For each primitive, perform intersection test.
    *   **Mailbox Optimization**: To avoid redundant tests for primitives spanning multiple cells, assign each ray a unique ID. Store the ray ID and intersection data in a cache associated with the primitive. If the cache contains a valid intersection for the current ray ID, skip the test [S4, unofficial].
    *   **Correctness Check**: If an intersection is found, verify that the intersection point lies within the current cell's bounds. If the intersection point is outside the current cell (e.g., in a neighboring cell), continue advancing the ray to subsequent cells and test their primitives until the intersection point is within the current cell or a closer intersection is found [S1, S4, unofficial].
2.  **Advance Ray**:
    *   Compare $tMaxX, tMaxY, tMaxZ$.
    *   Advance the ray to the nearest cell boundary:
        ```python
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
        ```
    [S1, unofficial]
3.  **Termination**:
    *   Stop if the ray exits the grid bounds ($X, Y, Z$ out of range) or if a valid intersection is found and no closer intersection is possible [S1, S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_regular_grid(primitives, p1, p2, gridsize):
    # Initialize grid
    num_cells = gridsize * gridsize * gridsize
    grid = [None] * num_cells
    
    # Calculate cell dimensions and reciprocals
    dx = (p2.x - p1.x) / gridsize
    dy = (p2.y - p1.y) / gridsize
    dz = (p2.z - p1.z) / gridsize
    dx_rec = 1.0 / dx
    dy_rec = 1.0 / dy
    dz_rec = 1.0 / dz
    
    # Map primitives to cells
    for prim in primitives:
        bound = prim.get_aabb()
        bv1 = bound.min
        bv2 = bound.max
        
        # Calculate cell range
        x1 = max(0, int((bv1.x - p1.x) * dx_rec))
        x2 = min(gridsize - 1, int((bv2.x - p1.x) * dx_rec) + 1)
        y1 = max(0, int((bv1.y - p1.y) * dy_rec))
        y2 = min(gridsize - 1, int((bv2.y - p1.y) * dy_rec) + 1)
        z1 = max(0, int((bv1.z - p1.z) * dz_rec))
        z2 = min(gridsize - 1, int((bv2.z - p1.z) * dz_rec) + 1)
        
        # Add primitive to intersected cells
        for x in range(x1, x2):
            for y in range(y1, y2):
                for z in range(z1, z2):
                    idx = x + y * gridsize + z * gridsize * gridsize
                    if grid[idx] is None:
                        grid[idx] = ObjectList()
                    grid[idx].add(prim)
                    
    return grid

def traverse_regular_grid(ray, grid, p1, gridsize):
    # Determine start cell
    if ray.origin is inside grid bounds:
        X = int((ray.origin.x - p1.x) / ((p2.x - p1.x) / gridsize))
        Y = int((ray.origin.y - p1.y) / ((p2.y - p1.y) / gridsize))
        Z = int((ray.origin.z - p1.z) / ((p2.z - p1.z) / gridsize))
    else:
        # Advance ray to grid entry point
        # If ray misses grid, return None
        entry_t, X, Y, Z = ray_intersect_grid(ray, p1, p2, gridsize)
        if entry_t is None:
            return None
            
    # Calculate deltas and initial tMax
    dx = (p2.x - p1.x) / gridsize
    dy = (p2.y - p1.y) / gridsize
    dz = (p2.z - p1.z) / gridsize
    
    stepX = 1 if ray.dir.x >= 0 else -1
    stepY = 1 if ray.dir.y >= 0 else -1
    stepZ = 1 if ray.dir.z >= 0 else -1
    
    # tMax is distance to next boundary
    # Boundary X is at p1.x + (X + (stepX>0?1:0)) * dx
    next_x = (X + (1 if stepX > 0 else 0)) * dx + p1.x
    next_y = (Y + (1 if stepY > 0 else 0)) * dy + p1.y
    next_z = (Z + (1 if stepZ > 0 else 0)) * dz + p1.z
    
    tMaxX = (next_x - ray.origin.x) / ray.dir.x
    tMaxY = (next_y - ray.origin.y) / ray.dir.y
    tMaxZ = (next_z - ray.origin.z) / ray.dir.z
    
    tDeltaX = dx / abs(ray.dir.x)
    tDeltaY = dy / abs(ray.dir.y)
    tDeltaZ = dz / abs(ray.dir.z)
    
    current_t = max(tMaxX, tMaxY, tMaxZ) # Actually, we start checking at the entry point
    
    # Traversal
    while 0 <= X < gridsize and 0 <= Y < gridsize and 0 <= Z < gridsize:
        # Process current cell
        cell_list = grid[X + Y * gridsize + Z * gridsize * gridsize]
        if cell_list is not None:
            for prim in cell_list:
                # Check mailbox to avoid redundant tests
                if not prim.is_tested_by(ray.id):
                    hit = prim.intersect(ray)
                    if hit is not None:
                        # Check if hit is within current cell bounds
                        # If hit point is outside current cell, continue traversal
                        # to ensure we find the closest valid hit or handle overlaps correctly
                        if hit.point is within cell_bounds(X, Y, Z, p1, dx, dy, dz):
                            return hit
                        else:
                            # Mark as tested to avoid re-testing in next cells
                            prim.mark_tested(ray.id)
                            # Continue to next cell to find the actual hit location
                            # or a closer hit
                            pass
                    else:
                        prim.mark_tested(ray.id)
        
        # Advance to next cell
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
                
    return None
```

### Edge Cases and Pitfalls
*   **Intersection Outside Cell**: A primitive stored in cell $F$ may be intersected by a ray while traversing cell $F$, but the intersection point lies outside cell $F$ (e.g., in cell $G$). Testing only the current cell misses this or requires redundant checks. The solution is to continue advancing the ray to subsequent cells and test their primitives until the intersection point is within the current cell or a closer intersection is found [S1, unofficial].
*   **Redundant Tests**: A single triangle may span multiple voxels. To avoid redundant intersection tests, use 'Mailboxes': assign each ray a unique number, store the ray number and intersection information in an object intersection cache, and skip tests if the cache already contains a valid intersection for that ray number [S4, unofficial].
*   **Boundary Conditions**: Primitives outside or on the boundary of the grid must be clamped to the grid indices $[0, GRIDSIZE-1]$ to prevent out-of-bounds access [S1, unofficial].
*   **Dynamic Scenes**: Not suitable for dynamic scenes if using Proximity Clouds acceleration, as pre-calculating distance information to the nearest full element is too costly [S5, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Build Time**: Linear with respect to the number of primitives or grid size, allowing creation from scratch and update at every frame with interactive performance for moderate sized scenes [S2].
*   **Memory Usage**: High for sparse scenes because the grid is uniform and cannot adapt to primitive density. Many triangles may cluster in a small area but occupy many empty voxels [S3, unofficial].
*   **Traversal Time**: Constant-time access to cells. However, traversal time can be higher than kd-trees due to the lack of adaptivity [S2].

### Measured Performance
*   On GPU, Regular Grids are considered the optimal structure for scenes with uniform data distribution due to high probability of cached texture accesses (up to 100x faster) [S5, unofficial].
*   Specific hardware/resolution metrics are not given in the sources.

### Strengths and Limitations
*   **Strengths**:
    *   Simple implementation [S3, unofficial].
    *   Constant-time access to elements and neighbors [S5, unofficial].
    *   Good data locality (successively accessed elements are spatially close in memory) [S5, unofficial].
    *   Fast build time, suitable for dynamic scenes [S2].
*   **Limitations**:
    *   High memory usage for sparse scenes [S3, unofficial].
    *   No adaptivity to scene complexity [S3, unofficial].
    *   Redundant intersection tests for primitives spanning multiple cells [S4, unofficial].
    *   Not suitable for secondary rays if the grid is too coarse or sparse [S1, unofficial].

### When to Use
*   Use for scenes with uniform data distribution, especially on GPU [S5, unofficial].
*   Use for dynamic scenes where fast rebuild is required [S2].
*   Do not use for sparse scenes or scenes with highly clustered primitives [S3, unofficial].

## Background

### Origin
*   Grid data structures were introduced by Gla84 [S2].
*   The 3D-DDA traversal algorithm is heavily based on the paper by Amanatides & Woo [S1, unofficial].

### History and Development
*   Regular grids have been prioritized since the first experiments of ray tracing on GPU [S5, unofficial].
*   Variations such as Nested Grids and Octrees were developed to address the lack of adaptivity by subdividing areas with high triangle density [S3, unofficial].
*   Proximity Clouds, a method to accelerate traversal in empty regions, was presented by Cohen and Sheffer [CS94] [S5, unofficial].

### Applications
*   Rendering analytical objects [S2].
*   Particle-based simulations [S2].
*   Computational fluids dynamics [S2].
*   Volume rendering [S2].

## Sources

[S1] flipcode - Raytracing Topics & Techniques, https://flipcode.com/archives/Raytracing_Topics_Techniques-Part_4_Spatial_Subdivisions.shtml, fetched 2026-10-24 (unofficial)
[S2] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S3] 光线追踪小记:空间加速结构Regular Grid与3DDDA, https://zhuanlan.zhihu.com/p/415869768, fetched 2026-10-25 (unofficial)
[S4] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
[S5] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
