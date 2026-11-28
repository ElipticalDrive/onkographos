# TomoSphero Raytracer

Also called: Spherical Grid Raytracer, TomoSphero

## Overview
TomoSphero is a differentiable raytracer designed for tomographic reconstruction on spherical grids, specifically targeting planetary and solar atmospheric modeling where spherical symmetry is prevalent [S1]. Unlike Cartesian ray-grid traversal methods that assume uniform voxel sizes and axis-aligned boundaries, TomoSphero explicitly computes intersections with spherical (radial), conical (elevational), and planar (azimuthal) boundaries to handle non-uniform and partial spherical domains [S1]. It solves the problem of efficiently computing line integrals (Fredholm integrals of the first kind) for optically thin media by precomputing voxel indices and intersection lengths, enabling fast GPU-accelerated forward and backward passes for gradient-based optimization [S1].

## How it works
The method discretizes the continuous line integral of an object $\rho$ along a ray into a weighted sum of voxel values, where weights are the intersection lengths of the ray with each voxel [S1]. The algorithm operates in four distinct phases: intersection computation, region indexing, sorting/length calculation, and integration [S1].

### Inputs and Outputs
**Inputs:**
*   **Ray:** Origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1].
*   **Grid:** Defined by shape $(N_r, N_e, N_a)$ representing radial, elevational, and azimuthal dimensions [S1].
    *   Radial boundaries: $N_r + 1$ spheres [S1].
    *   Elevational boundaries: $N_e + 1$ cones [S1].
    *   Azimuthal boundaries: $N_a + 1$ planes [S1].
*   **Object:** 3D volume $\rho$ defined on the spherical grid [S1].

**Outputs:**
*   **Measurements:** Scalar projection values for each ray [S1].
*   **Operator Data:** Arrays of voxel indices (`regs`) and intersection lengths (`lens`) for use in reconstruction [S1].

### Data Structures
The core data structure is a table (list of rows) where each row corresponds to a segment of the ray intersecting a specific voxel [S1]. For a single ray, the table has $N$ rows, where $N = 2(N_r + 1) + 2(N_e + 1) + (N_a + 1)$ [S1]. Each row contains:
1.  **Voxel Indices:** A tuple $(r_i, e_i, a_i)$ representing the radial, elevational, and azimuthal indices of the voxel [S1].
2.  **Intersection Length:** $\Delta l_i$, the distance along the ray within that voxel [S1].

The implementation stores these as parallel arrays accessible via `.regs` (indices) and `.lens` (lengths) [S1].

### Numbered Steps

**Step 1: Compute Ray-Boundary Intersections**
For each boundary type, compute the intersection points of the ray with all boundaries and their distances $l$ from the ray origin $\mathbf{o}$ [S1].
*   **Radial (Spheres) and Elevational (Cones):** Assume an upper bound of $2(N_r + 1)$ intersections for spheres/cones and $N_a + 1$ for planes [S1].
*   **Azimuthal (Planes):** Each ray intersects each of the $N_a + 1$ planar boundaries at most once [S1].
*   **No Intersection Handling:** If a boundary does not intersect the ray, store the coordinate but set the distance $l$ to $\infty$ [S1].

**Step 2: Determine Crossing Direction and Region Indices**
For each intersection point, determine if the ray crosses the boundary in a "positive" (increasing index) or "negative" (decreasing index) direction [S1].
*   The direction depends on the crossing point coordinate and the ray direction $\mathbf{d}$ [S1].
*   Convert boundary indices to region indices: If the crossing direction is negative, decrement the boundary index to get the region index [S1].

**Step 3: Sort and Compute Intersection Lengths**
1.  Collect all region indices from Step 2 into a single list [S1].
2.  Sort all intersection points by their distance $l$ from the ray start [S1].
3.  Compute the intersection length $\Delta l_i$ for each segment as the difference between adjacent sorted distances: $\Delta l_i = l_{i+1} - l_i$ [S1].
    *   Note: If $l_i = \infty$, $\Delta l_i$ becomes $\infty$ or $\text{NaN}$ [S1].
4.  Construct the 3D voxel index table:
    *   Start with the voxel index of the ray origin [S1].
    *   Insert the sorted region indices into the table [S1].
    *   Fill in missing indices from the previous row to form complete 3D tuples $(r, e, a)$ [S1].
    *   The total number of rows is $N = 2(N_r + 1) + 2(N_e + 1) + (N_a + 1)$ [S1].
5.  **Invalid Row Handling:** Set $\Delta l_i = 0$ for any row where:
    *   The region index is invalid (less than 0 or $\ge$ the grid dimension size) [S1].
    *   $\Delta l_i$ is $\infty$ or $\text{NaN}$ [S1].
    *   This ensures invalid segments contribute zero to the final sum [S1].

**Step 4: Line Integration**
Compute the projection measurement $y$ as the inner product of the object values and the intersection lengths [S1]:
$$
y = \sum_{i=1}^{N} \rho(r_i, e_i, a_i) \cdot \Delta l_i
$$
where $\rho(r_i, e_i, a_i)$ is the value of the object at the voxel indices $(r_i, e_i, a_i)$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def intersect_sphere(o, d, radius):
    """Standard ray-sphere intersection. Returns list of distances or [inf]."""
    oc = o
    b = 2.0 * sum(d[i] * oc[i] for i in range(3))
    c = sum(oc[i]**2 for i in range(3)) - radius**2
    disc = b**2 - 4.0 * c
    if disc < 0: return [float('inf')]
    sqrt_disc = math.sqrt(disc)
    t1 = (-b - sqrt_disc) / 2.0
    t2 = (-b + sqrt_disc) / 2.0
    return [t for t in (t1, t2) if t >= 0] if t1 >= 0 else ([t2] if t2 >= 0 else [float('inf')])

def intersect_cone(o, d, apex, axis, half_angle):
    """Standard ray-cone intersection. Returns list of distances or [inf]."""
    # Simplified standard quadratic solver for ray-cone [G]
    v = [o[i] - apex[i] for i in range(3)]
    cos_a = math.cos(half_angle)
    a = sum(d[i]*axis[i] for i in range(3))**2 - cos_a**2 * sum(d[i]**2 for i in range(3))
    b = 2.0 * (sum(d[i]*axis[i] for i in range(3)) * sum(v[i]*axis[i] for i in range(3)) - cos_a**2 * sum(v[i]*d[i] for i in range(3)))
    c = sum(v[i]*axis[i] for i in range(3))**2 - cos_a**2 * sum(v[i]**2 for i in range(3))
    disc = b**2 - 4.0 * a * c
    if disc < 0 or abs(a) < 1e-9: return [float('inf')]
    sqrt_disc = math.sqrt(disc)
    t1 = (-b - sqrt_disc) / (2.0 * a)
    t2 = (-b + sqrt_disc) / (2.0 * a)
    return [t for t in (t1, t2) if t >= 0] if t1 >= 0 else ([t2] if t2 >= 0 else [float('inf')])

def intersect_plane(o, d, point, normal):
    """Standard ray-plane intersection. Returns list of distances or [inf]."""
    denom = sum(d[i] * normal[i] for i in range(3))
    if abs(denom) < 1e-9: return [float('inf')]
    t = sum((point[i] - o[i]) * normal[i] for i in range(3)) / denom
    return [t] if t >= 0 else [float('inf')]

def get_crossing_direction(dist, ray_dir, boundary_type, boundary_idx, grid):
    """Determines if crossing increases or decreases the region index [G]."""
    # Standard logic: positive if moving towards higher index, negative otherwise [G]
    return 'positive' if ray_dir[0] > 0 else 'negative'

def get_voxel_index(origin, grid):
    """Maps ray origin to initial voxel indices [G]."""
    return (0, 0, 0)  # Placeholder for coordinate-to-index mapping [G]

def raytrace(ray_origin, ray_direction, grid, object_volume):
    # Step 1: Compute intersections
    r_dists = []
    for r_bound in grid.radial_boundaries:
        r_dists.extend(intersect_sphere(ray_origin, ray_direction, r_bound))
    e_dists = []
    for e_bound in grid.elevational_boundaries:
        e_dists.extend(intersect_cone(ray_origin, ray_direction, e_bound.apex, e_bound.axis, e_bound.half_angle))
    a_dists = []
    for a_bound in grid.azimuthal_boundaries:
        a_dists.extend(intersect_plane(ray_origin, ray_direction, a_bound.point, a_bound.normal))
        
    # Step 2: Determine crossing direction and region indices
    regions = []
    for i, dist in enumerate(r_dists):
        if dist == float('inf'): continue
        direction = get_crossing_direction(dist, ray_direction, 'radial', i, grid)
        region_idx = i if direction == 'positive' else i - 1
        regions.append(('r', region_idx, dist))
    for i, dist in enumerate(e_dists):
        if dist == float('inf'): continue
        direction = get_crossing_direction(dist, ray_direction, 'elevational', i, grid)
        region_idx = i if direction == 'positive' else i - 1
        regions.append(('e', region_idx, dist))
    for i, dist in enumerate(a_dists):
        if dist == float('inf'): continue
        direction = get_crossing_direction(dist, ray_direction, 'azimuthal', i, grid)
        region_idx = i if direction == 'positive' else i - 1
        regions.append(('a', region_idx, dist))
        
    # Step 3: Sort and compute lengths
    regions.sort(key=lambda x: x[2])
    segments = []
    for i in range(len(regions) - 1):
        dist_curr = regions[i][2]
        dist_next = regions[i+1][2]
        length = dist_next - dist_curr if dist_curr != float('inf') and dist_next != float('inf') else float('inf')
        segments.append({'type': regions[i][0], 'region_idx': regions[i][1], 'length': length})
        
    # Build 3D voxel table by propagating indices
    current_voxel = list(get_voxel_index(ray_origin, grid))
    voxel_table = []
    prev_dist = 0.0
    for seg in segments:
        dim = seg['type']
        idx = seg['region_idx']
        if dim == 'r': current_voxel[0] = idx
        elif dim == 'e': current_voxel[1] = idx
        elif dim == 'a': current_voxel[2] = idx
        voxel_table.append({'voxel': tuple(current_voxel), 'length': seg['length']})
        
    # Validate and zero invalid rows
    for entry in voxel_table:
        r, e, a = entry['voxel']
        length = entry['length']
        if (r < 0 or r >= grid.N_r or e < 0 or e >= grid.N_e or a < 0 or a >= grid.N_a):
            entry['length'] = 0.0
            continue
        if math.isinf(length) or math.isnan(length):
            entry['length'] = 0.0
            
    # Step 4: Line Integration
    total = 0.0
    for entry in voxel_table:
        r, e, a = entry['voxel']
        length = entry['length']
        rho = object_volume[r, e, a]
        total += rho * length
    return total
```

### Parameters
*   **Grid Shape $(N_r, N_e, N_a)$:** Determines the resolution and number of boundaries [S1].
*   **Data Types:** The implementation uses `int64` for indices and `float64` for distances and values to ensure precision [S1].

### Edge Cases and Pitfalls
*   **No Intersection:** If a boundary does not intersect the ray, the distance is set to $\infty$, and the corresponding segment is handled in validation [S1].
*   **Parallel Boundaries:** Numerically degenerate cases, such as a line of sight parallel to an azimuthal boundary plane, must be tested for consistency [S1].
*   **Memory Limits:** The algorithm does not support out-of-core operation [S1]. If the simultaneous computation for all rays exceeds available memory (VRAM), the algorithm fails [S1].
*   **Invalid Indices:** Rays passing outside the grid or crossing boundaries with no intersection result in invalid region indices, which are zeroed out to prevent contribution to the sum [S1].

## Operating characteristics
*   **Time Complexity:** The algorithm involves computing intersections for all boundaries ($O(N_r + N_e + N_a)$ per ray), sorting them ($O(K \log K)$ where $K$ is the number of intersections), and integrating ($O(K)$) [S1]. Since $K \approx 2(N_r + 1) + 2(N_e + 1) + (N_a + 1)$, the complexity is dominated by sorting and intersection computation [S1].
*   **Memory Complexity:** Peak memory usage is approximately $(2N_r + 2N_e + N_a) \cdot 6 \cdot N_{rays} \cdot 8 / 10^9$ GB, accounting for 6 arrays per ray (voxel indices for r/e/a, lengths, values, and temporary arrays) using 8-byte types [S1].
*   **Performance:** Validation tests on a unit sphere with a $(50, 50, 50)$ grid showed measurements within $0.00001\%$ of the true value, approaching machine epsilon for `float32` [S1].
*   **Strengths:** Efficient for spherical symmetries; differentiable via PyTorch; supports arbitrary projection geometries (cone, parallel) [S1].
*   **Limitations:** High memory usage for large grids and many rays; no out-of-core support; requires precomputation of intersections [S1].

## Background
TomoSphero was introduced to address the limitations of Cartesian discretization in tomographic reconstruction for planetary and solar atmospheres, where spherical symmetry is common [S1]. It is implemented in PyTorch to leverage GPU acceleration and automatic differentiation for gradient-based optimization [S1]. The method is used in atmospheric tomography, such as modeling the Carruthers Geocorona Observatory surveying Earth's exosphere [S1].

## Sources
[S1] TomoSphero: Fast Differentiable Projector for Planetary and Solar Tomography on Spherical Grids, https://doi.org/10.3847/1538-4357/ae6db5, fetched 2026-10-30
