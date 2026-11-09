# Bounding Volume Optimization

Also called: Adaptive Voxel-Array-Größe, BV Optimisation, Bounding Volume Optimisation, Bounding-Volume-Optimierung, SDF Bounding Volume Culling, Volumen-Größenanpassung

## Overview
Bounding Volume Optimization reduces the computational cost of volume rendering and ray marching by adapting the spatial extent of the voxel data or culling distant objects via bounding volumes [S1, unofficial][S2]. It solves the problem of processing empty space or irrelevant geometry, which causes unnecessary memory bandwidth usage and false positives during traversal [S1, unofficial]. Unlike sibling methods that compress data density (e.g., [Binary Voxel RLE Encoding](1071-binary-voxel-rle-encoding.md)) or smooth fields ([3D Voxel Grid Smoothing](833-3d-voxel-grid-smoothing.md)), this method focuses on minimizing the *domain* of computation by aligning the active voxel array dimensions with the actual data extents or by skipping SDF evaluations for objects outside their bounding volumes [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel array (or SDF primitives) defined on a discrete domain $(i, j, k)$ with initial dimensions $(N_x, N_y, N_z)$ [G].
*   **Output:** A reduced active bounding box defined by min/max coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$, or a list of bounding volumes for culling [S1, unofficial][S2].

**Data Structures:**
1.  **Voxel Grid:** A 3D array of scalar values (e.g., density or SDF values).
2.  **Bounding Box (AABB):** Defined by two corner vectors $\mathbf{b}_{min} = (x_{min}, y_{min}, z_{min})$ and $\mathbf{b}_{max} = (x_{max}, y_{max}, z_{max})$ in voxel indices or world space [S1, unofficial].
3.  **Primitive Bounding Volumes:** For SDF-based rendering, each primitive $P_i$ has an associated AABB $\mathbf{B}_i$ [S2].

**Steps:**
1.  **Analyze Extents:** Determine the minimum and maximum occupied indices $(i, j, k)$ in the voxel data or the world-space bounds of the SDF primitives [S1, unofficial][S2].
2.  **Compute Active Bounds:**
    *   For voxel arrays: Calculate $\mathbf{b}_{min}$ and $\mathbf{b}_{max}$ such that all non-empty voxels are contained within, and empty border regions are excluded [S1, unofficial].
    *   For SDFs: Compute the AABB for each primitive $P_i$ [S2].
3.  **Cull/Evaluate:**
    *   **Voxel Traversal:** Restrict ray marching steps to the range $[\mathbf{b}_{min}, \mathbf{b}_{max}]$. Rays starting outside or exiting this box terminate early [S1, unofficial].
    *   **SDF Evaluation:** For a ray at position $\mathbf{r}$, check if $\mathbf{r}$ is inside $\mathbf{B}_i$. If not, skip the evaluation of the expensive SDF function $S_i(\mathbf{r})$ [S2].
4.  **Update:** Apply the reduced bounds to the rendering loop or acceleration structure traversal [S1, unofficial][S2].

**Formulas:**
*   **Voxel Domain Reduction:**
    $$ N'_{x} = x_{max} - x_{min} + 1 $$
    $$ N'_{y} = y_{max} - y_{min} + 1 $$
    $$ N'_{z} = z_{max} - z_{min} + 1 $$
    Where $N'$ are the new dimensions, reducing memory load from $N_x N_y N_z$ to $N'_x N'_y N'_z$ [S1, unofficial].
*   **SDF Culling Check:**
    $$ \text{if } \mathbf{r} \notin \mathbf{B}_i \implies \text{skip } S_i(\mathbf{r}) $$
    This minimizes the cost of the map function by avoiding evaluations for objects that do not contribute to the minimal distance field [S2].

**Parameters:**
*   **Tolerance:** A small margin may be added to $\mathbf{b}_{min}/\mathbf{b}_{max}$ to account for anti-aliasing or interpolation, though not explicitly defined in sources [G].
*   **Resolution:** Performance gains depend on the ratio of occupied volume to total volume. In a 512³ grid with height 321, reducing to 512x321x512 saves significant memory [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **False Positives:** Using a fixed large array (e.g., 512³) for small data causes false positives during traversal, increasing cost [S1, unofficial].
*   **Hardware Behavior:** Modern GPUs are MIMD (Multiple Instruction, Multiple Data). Optimization helps by localizing expensive computations, allowing cheaper threads to finish early and distributing work across Streaming Multiprocessors (SMs) [S2].
*   **Lighting Artifacts:** In voxel rendering, AABBs can cause hard lighting edges because face normals are axis-aligned, leading to physically implausible shading [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_voxel_bounds(voxel_grid):
    """
    Finds the min/max indices of non-empty voxels.
    voxel_grid: 3D array of shape (Nx, Ny, Nz).
    Returns: (xmin, ymin, zmin), (xmax, ymax, zmax)
    """
    # Find indices where voxel value is non-zero (or above threshold)
    occupied = np.argwhere(voxel_grid > 0)
    
    if len(occupied) == 0:
        return None, None

    # Calculate min and max for each axis
    min_coords = occupied.min(axis=0)
    max_coords = occupied.max(axis=0)
    
    return min_coords, max_coords

def raymarch_with_bv_optimization(ray_origin, ray_direction, voxel_bounds, voxel_grid):
    """
    Marches a ray only within the optimized bounding volume.
    """
    t_min, t_max = 0.0, 1000.0 # Initial ray bounds
    
    # Intersect ray with voxel bounding box
    # (Standard AABB-ray intersection logic)
    hit_t_min, hit_t_max = intersect_aabb(ray_origin, ray_direction, voxel_bounds)
    
    if hit_t_min is None:
        return 0.0 # No hit
    
    # Update ray traversal range to the bounding box
    t_start = max(0.0, hit_t_min)
    t_end = hit_t_max
    
    density = 0.0
    t = t_start
    
    while t < t_end:
        # Sample voxel at current t
        pos = ray_origin + t * ray_direction
        val = sample_voxel(pos, voxel_grid)
        
        # Accumulate density (simplified)
        density += val
        
        t += step_size
        
    return density

def sdf_cull_primitives(ray_origin, ray_direction, primitives):
    """
    Skips SDF evaluation for primitives outside their bounding volumes.
    """
    min_dist = infinity
    
    for prim in primitives:
        # Check if ray intersects primitive's bounding volume
        if not ray_aabb_intersect(ray_origin, ray_direction, prim.bounding_box):
            continue # Skip expensive SDF evaluation
            
        # Evaluate SDF only if inside BV
        dist = prim.sdf_function(ray_origin, ray_direction)
        min_dist = min(min_dist, dist)
        
    return min_dist
```

## Subtopics

### Voxel Domain Culling
Reduces the 3D array dimensions to the tightest AABB containing the data. Choose when memory bandwidth is the bottleneck and data is sparse. Link: [Voxel Grid Optimization](4131-voxel-grid-optimization.md).

### SDF Primitive Culling
Uses bounding volumes to skip SDF evaluations for distant primitives. Choose when rendering scenes with many complex SDF objects. Link: [SDF Bounding Volume Culling](1071-binary-voxel-rle-encoding.md) (conceptual sibling).

## Operating characteristics
*   **Time Complexity:** Reduces the number of voxel samples or SDF evaluations from $O(N^3)$ or $O(P \cdot N)$ to $O(N'_{vol})$ or $O(P_{visible} \cdot N)$, where $P$ is primitives and $N$ is samples [S1, unofficial][S2].
*   **Memory Complexity:** Reduces memory footprint from $N_x N_y N_z$ to $N'_x N'_y N'_z$ [S1, unofficial].
*   **Measured Performance:** In a scene with 20 expensive primitives at 720p, enabling BV optimization reduced average frame duration from 41 ms (24 fps) to 21 ms (48 fps) [S2].
*   **Strengths:** Eliminates false positives; reduces memory load; localizes expensive computations [S1, unofficial][S2].
*   **Limitations:** AABBs can cause hard lighting artifacts in voxel rendering [S1, unofficial]. Requires pre-computation of bounds.
*   **When to Use:** When the scene contains large empty regions or many distant objects. When GPU thread divergence is high due to uniform cost distribution [S2].

## Background
Originating from voxel rendering and ray marching optimization techniques, this method addresses the inefficiency of fixed-size grids and uniform SDF evaluation [S1, unofficial][S2]. It is used in real-time raymarching engines and voxel-based physics simulations, where separating game worlds from entities allows dense voxel models to serve as bounding boxes for rigid bodies [S1, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
