# Grid-based Traversal

Also called: Hierarchical grid traversal for irregular volumes, Parker's brute-force approach, Parker's method

## Overview
Grid-based Traversal is a method for rendering irregular volume data (tetrahedral meshes) at interactive rates by partitioning the volume into a hierarchical grid structure to quickly determine candidate tetrahedra [S1][S2, unofficial]. It solves the performance bottleneck of testing all tetrahedra against every ray by spatially indexing the mesh, allowing for efficient culling and traversal. Unlike sibling methods that rely on complex connectivity or sorting, this approach uses a brute-force attachment of tetrahedra to grid cells, making it robust for unstructured data but potentially memory-intensive for highly non-uniform distributions.

## How it works
**Inputs:**
- A tetrahedral mesh $\mathcal{T}$ consisting of $N$ tetrahedra.
- A hierarchical rectilinear grid structure $\mathcal{G}$ (typically multilevel, e.g., 3 layers) covering the volume bounds [S1][S2, unofficial].
- Ray origin $O$ and direction $D$ for each pixel.
- An iso-value $v_{iso}$ for surface extraction [S1].

**Outputs:**
- Intersection points and interpolated scalar values for the iso-surface.

**Data Structures:**
1.  **Hierarchical Grid $\mathcal{G}$:** A multi-resolution grid. Each leaf cell $C_i$ stores:
    -   `min_val`: Minimum scalar value of all tetrahedra attached to this cell.
    -   `max_val`: Maximum scalar value of all tetrahedra attached to this cell.
    -   `tet_list`: A list of indices of all tetrahedra intersecting this cell.
2.  **Tetrahedron $T_j$:** Defined by vertices $\{V_0, V_1, V_2, V_3\}$ and associated scalar values $\{s_0, s_1, s_2, s_3\}$ [S1][S2, unofficial].

**Steps:**

1.  **Grid Construction (Preprocessing):**
    -   Initialize the hierarchical grid $\mathcal{G}$ with the volume bounds.
    -   For each tetrahedron $T_j$, determine all grid cells it intersects.
    -   Attach $T_j$ to the `tet_list` of every intersecting grid cell.
    -   Compute `min_val` and `max_val` for each grid cell based on the scalar values of its attached tetrahedra [S1][S2, unofficial].

2.  **Ray Traversal:**
    -   For each ray $(O, D)$, traverse the hierarchical grid $\mathcal{G}$ to find leaf cells intersected by the ray.
    -   Use `min_val` and `max_val` at each grid node to cull regions where the iso-value $v_{iso}$ cannot exist (i.e., if $v_{iso} < \text{min\_val}$ or $v_{iso} > \text{max\_val}$) [S1][S2, unofficial].

3.  **Tetrahedron Testing:**
    -   For each intersected leaf cell $C_i$, iterate through its `tet_list`.
    -   Since no connectivity information is used, all attached tetrahedra must be tested sequentially against the ray [S1].
    -   Perform ray-tetrahedron intersection tests.

4.  **Iso-surface Intersection:**
    -   If a ray intersects a tetrahedron face (triangle), compute the barycentric coordinates of the intersection point relative to the triangle vertices [S1].
    -   Use these barycentric coordinates to linearly interpolate the scalar value $s_{interp}$ at the intersection point from the vertex scalar values [S1].
    -   Compute the precise intersection with the iso-surface using linear interpolation of the scalar values along the ray segment within the tetrahedron [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_based_traversal(ray_origin, ray_dir, grid, tetrahedra):
    # 1. Traverse grid to find candidate leaf cells
    candidate_cells = traverse_grid(ray_origin, ray_dir, grid)
    
    intersections = []
    
    for cell in candidate_cells:
        # 2. Cull using min/max values
        if not (cell.min_val <= ray_iso_value <= cell.max_val):
            continue
            
        # 3. Test all attached tetrahedra
        for tet_idx in cell.tet_list:
            tet = tetrahedra[tet_idx]
            
            # Check ray-tetrahedron intersection
            hit, t_hit, face_normal, barycentric = ray_intersect_tet(ray_origin, ray_dir, tet)
            
            if hit:
                # 4. Interpolate scalar value at intersection
                s_interp = interpolate_scalar(tet, barycentric)
                
                # Compute precise iso-surface intersection if needed
                # This involves solving for t where scalar(t) == v_iso
                t_iso = compute_iso_intersection(ray_origin, ray_dir, tet, s_interp, ray_iso_value)
                
                if t_iso is not None:
                    intersections.append((t_iso, s_interp))
                    
    return sort_by_depth(intersections)

def interpolate_scalar(tet, barycentric):
    # Linear interpolation using barycentric coordinates (w0, w1, w2, w3)
    # and vertex scalar values (s0, s1, s2, s3)
    return (barycentric.w0 * tet.s0 + 
            barycentric.w1 * tet.s1 + 
            barycentric.w2 * tet.s2 + 
            barycentric.w3 * tet.s3)
```

**Edge Cases and Pitfalls:**
-   **Precision:** Linear interpolation assumes linear variation of scalars within the tetrahedron. High-frequency data may suffer from artifacts [S1].
-   **Boundaries:** Curvilinear volumes are not directly supported; hexahedra must be decomposed into tetrahedra (e.g., 5 tetrahedra per hexahedron) before indexing [S1].
-   **Performance:** The brute-force nature means that if many tetrahedra overlap a single grid cell, performance degrades. The hierarchical structure helps, but cell size must be tuned [S1][S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Grid traversal is $O(\log N)$ or $O(1)$ depending on implementation, but tetrahedron testing is $O(K)$ where $K$ is the number of tetrahedra per cell. In the worst case, $K$ can be large.
-   **Memory Complexity:** $O(N + C)$, where $N$ is the number of tetrahedra and $C$ is the number of grid cells. Memory usage is high because tetrahedra are stored in every cell they intersect.
-   **Measured Performance:** Parker et al. rendered a bioelectric field with over one million tetrahedra at interactive rates using a 512x512 viewport on 16 or more processors [S1][S2, unofficial].
-   **Strengths:** Simple to implement; handles unstructured data well; no connectivity required; scalable on parallel systems [S1][S2, unofficial].
-   **Limitations:** High memory consumption; inefficient for highly non-uniform tetrahedral distributions; curvilinear data requires preprocessing [S1].

## Background
Grid-based Traversal was introduced by Parker et al. [PPL* 99] as one of the first methods to render irregular volume data at interactive rates on a supercomputer [S1][S2, unofficial]. It was developed to address the limitations of surface ray tracers on unstructured grids by using a hierarchical rectilinear grid to accelerate tetrahedron selection [S1][S2, unofficial]. The method was demonstrated on bioelectric field simulations, showing that ray tracing systems scale well with the number of processors [S1][S2, unofficial].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
