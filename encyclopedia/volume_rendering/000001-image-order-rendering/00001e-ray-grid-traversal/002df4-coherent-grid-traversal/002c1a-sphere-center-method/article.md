# Sphere-Center Method
Also called: enlarged frustum traversal for particles
## Overview
The Sphere-Center Method is an optimization for glyph-based particle visualization within the Coherent Grid Traversal (CGT) framework. It solves the problem of redundant storage and intersection testing for spherical primitives that overlap multiple grid cells by storing each sphere in exactly one cell—specifically, the cell containing its center. Unlike traditional grid traversal which requires mailboxes or data duplication to handle overlapping primitives, this method eliminates indirection overhead by traversing an enlarged frustum. It is distinct from [Macrocell Hierarchical CGT](5464-macrocell-hierarchical-cgt.md) and [Bundle Parallelisation](6293-bundle-parallelisation.md) by relying on geometric expansion of the traversal volume rather than hierarchical skipping or packet-level parallelism to achieve efficiency.

## How it works
The method exploits the properties of particle-based simulation datasets where radii are uniform or fall within a well-defined range, and particles are uniformly distributed or densely packed. It uses a grid as the acceleration structure, constructed on-the-fly for time-varying data.

**Inputs and Outputs:**
*   **Input:** A set of particles, each defined by a center position $C_i$ and radius $r_i$. A ray packet bounding frustum $F$.
*   **Output:** A list of sphere intersections for the ray packet.

**Data Structures:**
*   **Grid:** A 3D grid where each cell stores a list of sphere indices.
*   **Storage Layout:** Each sphere is stored in exactly one cell: the cell $(x, y, z)$ such that the sphere's center $C_i$ lies within that cell's bounds. No duplicate entries are created for spheres that geometrically overlap adjacent cells.

**Parameters:**
*   $r_{max}$: The maximum radius across all particles in the dataset. If radii vary, this is the upper bound. If uniform, $r_{max} = r_i$.

**Algorithm Steps:**

1.  **Determine Maximum Radius:** Compute $r_{max} = \max_i(r_i)$.
2.  **Compute Enlarged Frustum:** Construct an enlarged frustum $F_r$ from the original ray packet frustum $F$.
    *   For each bounding plane of $F$ with normal direction vector $dU$, shift the plane outward by distance $s$.
    *   The shift distance $s$ is calculated as:
        $$s = r_{max} \sqrt{1 + dU^2}$$
        where $dU$ is the direction vector of the given plane's normal. Note: In standard Cartesian coordinates where normals are unit vectors, $dU^2 = 1$, simplifying the term to $\sqrt{2}$. However, the source formula uses $dU^2$ explicitly, implying $dU$ may be a non-normalized direction component or the formula accounts for specific coordinate scaling. We retain the source's exact form.
    *   Shift the near and far planes of the frustum by $r_{max}$.
3.  **Grid Construction:** Build the grid on-the-fly. For each particle $i$ with center $C_i$:
    *   Determine the grid cell index $(cx, cy, cz)$ containing $C_i$.
    *   Append the sphere index $i$ to the list of objects in cell $(cx, cy, cz)$.
4.  **Traversal:** Traverse the grid using the enlarged frustum $F_r$.
    *   Identify all grid cells that intersect $F_r$.
    *   For each intersected cell, retrieve the list of spheres stored in that cell.
    *   Perform ray-sphere intersection tests for all rays in the packet against all spheres in the cell.
5.  **Early Termination Adjustment:** Adjust early ray termination criteria. Because the frustum is enlarged, a sphere whose center lies in the "next" slice (relative to the ray packet's original exit) might still intersect the ray. The termination condition must account for spheres in cells adjacent to the traversal boundary that were included due to the enlargement.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_center_method(particles, ray_packet):
    """
    particles: List of objects with attributes .center (vec3) and .radius (float)
    ray_packet: Object with attributes .frustum (list of planes) and .rays (list of rays)
    Returns: List of intersection results
    """
    # Step 1: Determine maximum radius
    r_max = 0.0
    for p in particles:
        if p.radius > r_max:
            r_max = p.radius

    # Step 2: Compute enlarged frustum
    # Original frustum planes are defined by normal n and distance d from origin
    # Plane equation: n . x + d = 0
    enlarged_planes = []
    for plane in ray_packet.frustum:
        n = plane.normal
        d = plane.distance
        # dU is the direction vector of the plane's normal. 
        # Assuming n is normalized, dU^2 = 1. If n is not normalized, use n directly.
        # The source formula: s = r_max * sqrt(1 + dU^2)
        # If n is unit vector, dU = n, dU^2 = 1. s = r_max * sqrt(2).
        # We implement the general case as stated.
        du_sq = dot(n, n) 
        s = r_max * sqrt(1.0 + du_sq)
        
        # Shift plane outward. For a plane n.x + d = 0, shifting by s along n
        # changes d to d - s (assuming n points outward from the frustum interior).
        # Note: Sign depends on convention. We assume standard outward normal.
        enlarged_planes.append(Plane(n, d - s))

    # Shift near and far planes by r_max
    # Assuming near/far planes are part of the frustum list and handled similarly
    # or treated as special cases. The source says "near and far planes ... shifted by r_max".
    # This implies s_near_far = r_max.
    # We assume the frustum list includes all 6 planes.
    # For near/far, the shift is simply r_max.
    # Re-evaluating: The formula s = r_max * sqrt(1 + dU^2) is for "bounding planes".
    # Near/far are also bounding planes. If their normals are axis-aligned, dU^2=1.
    # So the formula applies to all planes.
    
    # Step 3: Grid Construction
    grid = create_empty_grid()
    for i, p in enumerate(particles):
        cell = get_cell_index(p.center, grid.resolution, grid.origin)
        grid[cell].append(i)

    # Step 4: Traversal
    intersections = []
    cells_to_visit = get_cells_intersecting_frustum(grid, enlarged_planes)
    
    for cell in cells_to_visit:
        sphere_indices = grid[cell]
        for ray in ray_packet.rays:
            for idx in sphere_indices:
                p = particles[idx]
                hit = intersect_ray_sphere(ray, p.center, p.radius)
                if hit:
                    intersections.append(hit)

    # Step 5: Early Termination
    # Adjust termination to include spheres in cells adjacent to the original frustum
    # that were included in cells_to_visit due to enlargement.
    # This is handled implicitly by traversing the enlarged frustum cells.
    
    return intersections
```

**Edge Cases and Pitfalls:**
*   **Precision:** The shift calculation involves a square root. Ensure floating-point precision is sufficient for large $r_{max}$ or small grid cells.
*   **Boundary Conditions:** The enlargement means spheres whose centers are outside the original ray packet's frustum but whose bodies intersect it will be tested. This is correct behavior but increases the number of intersection tests compared to a tight fit.
*   **Degenerate Input:** If $r_{max} = 0$, the enlarged frustum equals the original frustum. The method degrades to standard point-in-cell storage.

## Subtopics

### Macrocell Hierarchical CGT
Uses a hierarchical grid structure to skip empty space faster than a flat grid. Choose when the scene has large empty regions and the overhead of hierarchy traversal is justified. See [Macrocell Hierarchical CGT](5464-macrocell-hierarchical-cgt.md).

### Bundle Parallelisation
Parallelizes the ray-sphere intersection tests across multiple rays in a packet using SIMD or GPU threads. Choose when hardware supports wide vector operations. See [Bundle Parallelisation](6293-bundle-parallelisation.md).

## Operating characteristics
*   **Time Complexity:** Grid construction is $O(N)$ where $N$ is the number of particles. Traversal is proportional to the number of cells intersecting the enlarged frustum. Intersection testing is proportional to the number of spheres stored in those cells.
*   **Memory Complexity:** $O(N)$ for storing particle indices in the grid. No duplicate storage for overlapping spheres.
*   **Performance:** The method reduces storage requirements and avoids the overhead of mailboxes and data reordering. It performs well for uniformly sized primitives. For time-varying data, on-the-fly grid construction is required, which adds CPU/GPU overhead.
*   **Strengths:** Eliminates redundant computation and storage. Simple implementation. Good locality of reference.
*   **Limitations:** Requires knowledge of $r_{max}$ beforehand. The enlarged frustum may include more cells than necessary if $r_{max}$ is much larger than typical radii. Not suitable for highly variable radii where $r_{max}$ is an outlier.
*   **When to use:** Particle visualization with uniform or narrowly distributed radii. Interactive ray tracing.
*   **When not to use:** Scenes with highly variable particle sizes where $r_{max}$ is significantly larger than the median radius, leading to excessive frustum enlargement.

## Background
The Sphere-Center Method was introduced in the context of interactive ray tracing for visualizing large particle datasets, such as those from material point method simulations. It builds on the Coherent Grid Traversal (CGT) algorithm [S1]. The approach is motivated by observations similar to offset surfaces used in collision detection. It addresses the limitations of traditional grids where primitives overlap multiple cells, requiring mailboxes or reordering. The method was demonstrated using results from structural mechanics simulations [S1].

## Sources
[S1] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
