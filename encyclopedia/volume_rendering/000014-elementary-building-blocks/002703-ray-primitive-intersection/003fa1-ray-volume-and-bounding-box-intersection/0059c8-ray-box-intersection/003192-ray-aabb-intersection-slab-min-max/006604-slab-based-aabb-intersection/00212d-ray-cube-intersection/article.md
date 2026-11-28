# Ray-Cube Intersection
Also called: Ray-Cube Slab Intersection, CubeCast, RayCubeSlabsIntersection
## Overview
This method calculates the precise entry ($t_0$) and exit ($t_1$) intersection points of a ray with an axis-aligned cube defined by lower and upper bounds [S1, unofficial]. It solves the problem of determining the exact boundary hit for ray-casting or ray-marching within voxel grids, providing the entry position and the exit face index required for subsequent traversal or shading [S1, unofficial]. Unlike [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md), which computes the interval $[t_{min}, t_{max}]$ for traversal logic, this method explicitly returns the 3D intersection coordinates and the specific face index exited, which is critical for hardware-accelerated voxel rendering where the first solid voxel must be identified [S1, unofficial].
## How it works
**Inputs:**
- Ray origin: $\mathbf{r}_{pos}$ (3D vector) [S1, unofficial]
- Ray direction: $\mathbf{r}_{dir}$ (3D vector) [S1, unofficial]
- Cube lower bound: $\mathbf{d}_{lower}$ (3D vector) [S1, unofficial]
- Cube upper bound: $\mathbf{d}_{upper}$ (3D vector) [S1, unofficial]

**Outputs:**
- Entry hit: Structure containing position $\mathbf{t}_0$ and entry face index $\mathbf{inface}$ [S1, unofficial]
- Exit hit: Structure containing position $\mathbf{t}_1$ and exit face index $\mathbf{outface}$ [S1, unofficial]

**Data Structures:**
- `Hit`: A structure containing:
  - `pos`: 3D vector ($\mathbf{t}_0$ or $\mathbf{t}_1$) [S1, unofficial]
  - `face`: Integer index representing the face normal direction [S1, unofficial]

**Lookup Tables:**
Two $2 \times 3$ integer lookup tables map the axis index and ray direction sign to a face normal index [S1, unofficial].
- `outfaces[axis][direction]`: Returns the exit face index. `direction` is 0 if $\mathbf{r}_{dir}[axis] < 0$ (negative face), else 1 (positive face) [S1, unofficial].
- `infaces[axis][direction]`: Returns the entry face index. `direction` is 0 if $\mathbf{r}_{dir}[axis] < 0$ (positive face), else 1 (negative face) [S1, unofficial].

**Steps for Exit Point ($t_1$):**
1. Compute the intersection parameter vector $\mathbf{Q}_v$ for the upper bound:
   $$ \mathbf{Q}_v = \frac{\mathbf{d}_{upper} - \mathbf{r}_{pos}}{\mathbf{r}_{dir}} $$
   Component-wise: $Q_v[i] = \frac{d_{upper}[i] - r_{pos}[i]}{r_{dir}[i]}$ for $i \in \{0, 1, 2\}$ [S1, unofficial].
2. Identify the axis index $i$ corresponding to the minimum component of $\mathbf{Q}_v$:
   $$ i = \text{argmin}(\mathbf{Q}_v) $$ [S1, unofficial]
3. Retrieve the scalar intersection distance $Q = Q_v[i]$ [S1, unofficial].
4. Calculate the 3D intersection point:
   $$ \mathbf{t}_1 = \mathbf{r}_{pos} + \mathbf{r}_{dir} \cdot Q $$ [S1, unofficial]
5. Determine the exit face $\mathbf{outface}$ using the lookup table `outfaces[i][direction]`, where `direction` is 0 if $\mathbf{r}_{dir}[i] < 0$ and 1 otherwise [S1, unofficial].
6. Clamp the intersection point $\mathbf{t}_1$ to the exact boundary of the cube by setting the $i$-th component of $\mathbf{t}_1$ to $\mathbf{d}_{upper}[i]$ [S1, unofficial].
7. Return the hit structure containing $\mathbf{t}_1$ and $\mathbf{outface}$ [S1, unofficial].

**Steps for Entry Point ($t_0$):**
1. Compute the intersection parameter vector $\mathbf{Q}_v$ for the lower bound:
   $$ \mathbf{Q}_v = \frac{\mathbf{d}_{lower} - \mathbf{r}_{pos}}{\mathbf{r}_{dir}} $$ [S1, unofficial]
2. Coalesce NaN values in $\mathbf{Q}_v$ to negative infinity to handle division by zero or invalid inputs [S1, unofficial].
3. Identify the axis index $i$ corresponding to the maximum component of $\mathbf{Q}_v$:
   $$ i = \text{argmax}(\mathbf{Q}_v) $$ [S1, unofficial]
4. Retrieve the scalar intersection distance $Q = Q_v[i]$ [S1, unofficial].
5. Calculate the 3D intersection point:
   $$ \mathbf{t}_0 = \mathbf{r}_{pos} + \mathbf{r}_{dir} \cdot Q $$ [S1, unofficial]
6. Determine the entry face $\mathbf{inface}$ using the lookup table `infaces[i][direction]`, where `direction` is 0 if $\mathbf{r}_{dir}[i] < 0$ and 1 otherwise [S1, unofficial].
7. Clamp the intersection point $\mathbf{t}_0$ to the exact boundary of the cube by setting the $i$-th component of $\mathbf{t}_0$ to $\mathbf{d}_{lower}[i]$ [S1, unofficial].
8. Return the hit structure containing $\mathbf{t}_0$ and $\mathbf{inface}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def min_component_index(v):
    min_val = v[0]
    min_idx = 0
    if v[1] < min_val:
        min_val = v[1]
        min_idx = 1
    if v[2] < min_val:
        min_idx = 2
    return min_idx

def max_component_index(v):
    max_val = v[0]
    max_idx = 0
    if v[1] > max_val:
        max_val = v[1]
        max_idx = 1
    if v[2] > max_val:
        max_idx = 2
    return max_idx

def safe_div(a, b):
    if b == 0.0:
        return float('inf') if a > 0 else float('-inf')
    return a / b

def ray_cube_intersection_exit(ray_pos, ray_dir, d_upper):
    # Step 1: Compute Qv
    qv = [safe_div(d_upper[i] - ray_pos[i], ray_dir[i]) for i in range(3)]
    
    # Pitfall: If any component of Qv is negative, the ray originates outside or points away.
    if any(q < 0.0 for q in qv):
        return None, None
        
    # Step 2: Identify axis index i for minimum component
    i = min_component_index(qv)
    
    # Step 3: Retrieve scalar Q
    Q = qv[i]
    
    # Step 4: Calculate 3D intersection point
    t1 = [ray_pos[j] + ray_dir[j] * Q for j in range(3)]
    
    # Step 5: Determine exit face
    # direction is 0 if ray_dir[i] < 0 (negative face), else 1 (positive face)
    direction = 0 if ray_dir[i] < 0 else 1
    # Lookup table outfaces[axis][direction]
    outfaces = [[0, 1], [2, 3], [4, 5]] # X-, X+, Y-, Y+, Z-, Z+
    outface = outfaces[i][direction]
        
    # Step 6: Clamp t1 to exact boundary
    t1[i] = d_upper[i]
    
    return t1, outface

def ray_cube_intersection_entry(ray_pos, ray_dir, d_lower):
    # Step 1: Compute Qv
    qv = [safe_div(d_lower[i] - ray_pos[i], ray_dir[i]) for i in range(3)]
            
    # Step 2: Coalesce NaN to -inf
    qv = [float('-inf') if math.isnan(q) else q for q in qv]
            
    # Pitfall: If all components of Qv are negative, the ray does not intersect.
    if all(q < 0.0 for q in qv):
        return None, None
            
    # Step 3: Identify axis index i for maximum component
    i = max_component_index(qv)
            
    # Step 4: Retrieve scalar Q
    Q = qv[i]
            
    # Step 5: Calculate 3D intersection point
    t0 = [ray_pos[j] + ray_dir[j] * Q for j in range(3)]
            
    # Step 6: Determine entry face
    # direction is 0 if ray_dir[i] < 0 (positive face), else 1 (negative face) per evidence
    direction = 0 if ray_dir[i] < 0 else 1
    # Lookup table infaces[axis][direction]
    infaces = [[1, 0], [3, 2], [5, 4]] # X+, X-, Y+, Y-, Z+, Z-
    inface = infaces[i][direction]
            
    # Step 7: Clamp t0 to exact boundary
    t0[i] = d_lower[i]
            
    return t0, inface
```
## Subtopics          

### [Ray-Cube Intersection (Brick/Slab)](3620-ray-cube-intersection-brick-slab.md)
This method performs a Digital Differential Analyzer (DDA) traversal within a single $4 \times 4 \times 4$ voxel brick to locate the first solid voxel intersected by a ray. It solves the problem of primary hit resolution in hardware-accelerated voxel rendering, where a coarse bounding volume intersection identifies a candidate brick, but the exact first solid voxel must be determined to handle transparency and fringe cases correctly. Unlike [In-Brick DDA for Occlusion](3623-in-brick-dda-for-occlusion.md), which visits all solid voxels to determine shadowing, this method stops at the first valid hit. It builds on [Ray-Cube Intersection](2466-ray-cube-intersection.md) by refining the entry point into a specific voxel index and material check.

## Operating characteristics
- **Time Complexity:** $O(1)$ per intersection, involving a fixed number of arithmetic operations and comparisons [G].
- **Memory Complexity:** $O(1)$, requiring only a few temporary vectors and lookup tables [G].
- **Strengths:** Provides exact boundary positions and face normals, which are necessary for precise voxel traversal and shading. Handles degenerate cases (ray parallel to axes) via NaN/Infinity coalescing [S1, unofficial].
- **Limitations:** Assumes the ray intersects the cube. If the ray originates inside or misses, the logic must be checked (negative $Q$). The method does not perform traversal, only intersection [S1, unofficial].
- **When to use:** When the exact entry/exit points and face normals are required, such as in voxel rendering pipelines or precise ray-casting [S1, unofficial].
- **When not to use:** When only the interval $[t_{min}, t_{max}]$ is needed for traversal logic; use [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md) instead [G].
## Background
The method originates from implementations in voxel rendering engines, specifically those dealing with brick-based traversal [S1, unofficial]. The specific terminology and lookup table approach are found in the `landscapes` repository [S1, unofficial] and the `r3d` library [S2, unofficial]. It is used in hardware-accelerated voxel rendering to resolve the first solid voxel within a coarse bounding volume [S1, unofficial].
## Sources
[S1] realazthat/landscapes, https://github.com/realazthat/landscapes, fetched 2026-10-26 (unofficial)
[S2] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
