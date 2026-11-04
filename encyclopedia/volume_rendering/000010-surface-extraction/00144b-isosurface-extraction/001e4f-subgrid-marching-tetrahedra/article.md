# Subgrid Marching Tetrahedra

Also called: Subgrid MT, single-tet reconstruction

## Overview
Subgrid Marching Tetrahedra (Subgrid MT) is an isosurface extraction algorithm that reconstructs an intersection-free triangulation by decomposing a regular cubic grid into tetrahedra and resolving the scalar field crossings within each tetrahedron [S1, S2, unofficial]. Unlike its sibling [Marching Cubes](4328-naive-isosurface-extraction.md), which suffers from topological ambiguities in specific configurations, this method guarantees a manifold, self-intersection-free surface by enforcing a consistent local construction policy based on edge intersection counts [S1, S2, unofficial]. It produces a mesh that matches voxel data more accurately than Marching Cubes, at the cost of a higher vertex count [S2, unofficial].

## How it works
Subgrid MT operates in two phases: global grid traversal to determine edge intersections, and local tetrahedral reconstruction to generate triangles. The method accepts three input types: a triangle mesh remeshed on a grid, an analytical Signed Distance Function (SDF), or precomputed edge intersections in `.npz` format [S2, unofficial].

### Data Structures

**Edge Coordinates (Single Tet):**
An array of six non-negative integers representing the number of times the isosurface crosses each edge of a single tetrahedron. The order is fixed as:
$$ \text{edge\_coords} = [e_{01}, e_{02}, e_{03}, e_{23}, e_{13}, e_{12}] $$
where $e_{ij}$ is the intersection count on the edge between vertex $i$ and vertex $j$ [S1, unofficial].

**Result Structures:**
1.  **`sgmt_result` (C Library):**
    *   `triangles`: `int32_t` array, 3 indices per triangle.
    *   `num_triangles`: Count of triangles.
    *   `num_edge_points`: Count of intersection points on edges.
    *   `num_steiner`: Count of Steiner points.
    *   `steiner_bary`: Array of 4 barycentric coordinates per Steiner point [S1, unofficial].
2.  **`PrimalResult` (Python):**
    *   `vertices`: $(N, 3)$ `float64` array.
    *   `faces`: List of index arrays (one per face, potentially n-gons) [S2, unofficial].
3.  **`DualResult` (Python):**
    *   Contains dual polygon mesh data via Quadric Error Function (QEF) solve [S2, unofficial].

### Algorithm Steps

1.  **Grid Construction and Intersection Detection:**
    *   If input is a mesh, weld vertices, triangulate, center/scale to $[-1, 1]^3$, and apply a rigid translation offset (magnitude 0.01, seed-dependent) to break axis-alignment [S2, unofficial].
    *   Construct an $n^3$ cubic grid, which decomposes into $5n^3$ tetrahedra [S2, unofficial].
    *   Compute edge intersections. For mesh input, use ray intersection or CGAL exact queries. For SDF, evaluate at grid vertices and interpolate. Store intersection counts and parameters ($t$-values) for all grid edges [S2, unofficial].
    *   Validate the "even-sum" condition: For each tetrahedron, the sum of intersections on its faces must be even. A non-zero count of "non-even tets" indicates potential non-watertightness or numerical artifacts [S2, unofficial].

2.  **Local Tetrahedral Reconstruction:**
    For each tetrahedron in the grid, invoke the reconstruction routine with its six edge intersection counts [S1, unofficial].
    
    *   **Input Validation:** Ensure exactly six non-negative integers are provided. If validation fails, abort [S1, unofficial].
    *   **Simplicial Embedding:** By default, use simplicial embedding (flag `simplicial=1`). This places Steiner points inside the tetrahedron to resolve ambiguities [S1, unofficial].
    *   **Curve Classification:** Identify boundary curves on the tetrahedron faces. Curves are classified as:
        *   **Open:** Curves that do not close within the tetrahedron.
        *   **Scoop:** Curves that wrap around a vertex.
        *   **Normal:** Standard intersection curves [S2, unofficial].
    *   **Triangulation:** Construct the local surface using one of three policies:
        *   **Primal:** Standard triangulation interpolating edge crossings.
        *   **Dual:** Solve a QEF for dual vertices to create a dual polygon mesh [S2, unofficial].
        *   **Greedy:** Fan triangulation from boundary curves [S2, unofficial].
    *   **Steiner Points:** If simplicial embedding is active, compute barycentric coordinates for internal Steiner points [S1, unofficial].

3.  **Vertex Merging and Output:**
    *   **Combinatorial Merge (Default):** Merge vertices based on edge identity to ensure manifold connectivity.
    *   **Numerical Merge:** If `mergeEPS >= 0`, merge vertices within a positional epsilon [S2, unofficial].
    *   **No Merge:** If `noMerge` is set, output a raw triangle soup [S2, unofficial].
    *   Iterate over the result triangles array in strides of three, outputting vertex indices [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subgrid_marching_tetrahedra(input_type, input_data, resolution=64, 
                                simplicial=True, merge_eps=-1, use_cgal=False):
    """
    Main pipeline for Subgrid Marching Tetrahedra.
    
    Args:
        input_type: 'mesh', 'sdf', or 'npz'
        input_data: Path to mesh/SDF name or .npz file
        resolution: Grid resolution n (creates n^3 grid)
        simplicial: Use simplicial embedding for ambiguity resolution
        merge_eps: Positional merge epsilon. -1 for combinatorial merge.
        use_cgal: Use CGAL for exact intersection queries (mesh input only)
        
    Returns:
        Result object containing vertices and faces
    """
    # 1. Preprocessing
    if input_type == 'mesh':
        mesh = load_and_weld_mesh(input_data)
        mesh = center_and_scale_to_unit_cube(mesh)
        mesh = apply_seed_offset(mesh, seed=1)
        # 2. Grid Intersection
        grid_edges, edge_intersections = compute_grid_intersections(
            mesh, resolution, use_cgal=use_cgal
        )
    elif input_type == 'sdf':
        grid_edges, edge_intersections = compute_sdf_intersections(
            input_data, resolution
        )
    elif input_type == 'npz':
        grid_edges, edge_intersections = load_precomputed_intersections(input_data)
    else:
        raise ValueError("Invalid input type")

    # 3. Local Reconstruction
    all_triangles = []
    all_vertices = []
    steiner_points = []
    non_even_tets = 0

    for tet_idx, tet_vertices in enumerate(grid_edges.tetrahedra):
        # Get edge intersection counts for this tet
        edge_counts = [edge_intersections[edge].count for edge in tet_edges]
        
        # Validate even-sum condition
        if not is_even_sum(edge_counts):
            non_even_tets += 1
            
        # Reconstruct local surface
        local_result = reconstruct_tet(
            edge_counts=edge_counts,
            tet_vertices=tet_vertices,
            simplicial=simplicial
        )
        
        # Merge local vertices with global indices
        merged_triangles, new_vertices = merge_vertices(
            local_result.triangles,
            local_result.vertices,
            all_vertices,
            merge_eps=merge_eps
        )
        
        all_triangles.extend(merged_triangles)
        all_vertices.extend(new_vertices)
        steiner_points.extend(local_result.steiner_bary)

    # 4. Output
    return PrimalResult(
        vertices=all_vertices,
        faces=all_triangles,
        non_even_tets=non_even_tets,
        steiner_points=steiner_points
    )

def reconstruct_tet(edge_counts, tet_vertices, simplicial=True):
    """
    Reconstructs a single tetrahedron's surface from edge coordinates.
    
    Args:
        edge_counts: List of 6 integers [e01, e02, e03, e23, e13, e12]
        tet_vertices: Array of 4 vertices (x, y, z)
        simplicial: Boolean flag for simplicial embedding
        
    Returns:
        sgmt_result with triangles, edge points, and steiner points
    """
    # Validate input
    if len(edge_counts) != 6 or any(c < 0 for c in edge_counts):
        raise ValueError("Invalid edge coordinates")
        
    # Classify curves
    open_curves, scoop_curves, normal_curves = classify_boundary_curves(
        edge_counts, tet_vertices
    )
    
    # Check for non-even tets
    if not is_even_sum(edge_counts):
        # Output may have holes or be non-orientable
        pass
        
    # Construct local surface
    if simplicial:
        # Place Steiner points and resolve ambiguities
        steiner_bary = compute_steiner_barycentric(edge_counts)
        triangles = build_simplicial_triangulation(
            edge_counts, tet_vertices, steiner_bary
        )
    else:
        triangles = build_standard_triangulation(
            edge_counts, tet_vertices
        )
        
    # Compute intersection points
    edge_points = compute_edge_intersections(edge_counts, tet_vertices)
    
    return sgmt_result(
        triangles=triangles,
        num_triangles=len(triangles),
        num_edge_points=len(edge_points),
        num_steiner=len(steiner_bary),
        steiner_bary=steiner_bary
    )

def compute_edge_intersections(edge_counts, tet_vertices):
    """
    Computes 3D coordinates of intersection points on edges.
    
    Args:
        edge_counts: List of 6 integers
        tet_vertices: Array of 4 vertices
        
    Returns:
        List of 3D points
    """
    points = []
    edge_pairs = [(0,1), (0,2), (0,3), (2,3), (1,3), (1,2)]
    
    for i, count in enumerate(edge_counts):
        v1, v2 = tet_vertices[edge_pairs[i]]
        for _ in range(count):
            # Interpolate t-value (simplified: uniform distribution for demo)
            # In practice, t is computed from scalar field values
            t = compute_t_value(v1, v2)
            point = lerp(v1, v2, t)
            points.append(point)
            
    return points

def lerp(v1, v2, t):
    """Linear interpolation between two 3D vectors."""
    return [v1[i] + t * (v2[i] - v1[i]) for i in range(3)]

def compute_t_value(v1, v2):
    """
    Computes the parameter t for intersection.
    Assumes scalar values f1, f2 are available from context.
    Formula: t = (I - f1) / (f2 - f1) where I is isovalue.
    """
    # Placeholder for actual scalar field evaluation
    # In a real implementation, f1 and f2 are passed or looked up
    f1 = 0.0 # Placeholder
    f2 = 1.0 # Placeholder
    isovalue = 0.5 # Placeholder
    if abs(f2 - f1) < 1e-9:
        return 0.5
    return (isovalue - f1) / (f2 - f1)
```

### Parameters and Flags

*   **`edge_coords[6]`**: Array of six non-negative integers. Order: $(e_{01}, e_{02}, e_{03}, e_{23}, e_{13}, e_{12})$ [S1, unofficial].
*   **`simplicial`**: Boolean flag. Default `1` (on). Enables simplicial embedding [S1, unofficial].
*   **`mergeEPS`**: Positional merge epsilon. Default `-1` (combinatorial merge). Values $\ge 0$ switch to numerical merge [S2, unofficial].
*   **`noMerge`**: Disables vertex merging, outputting a raw triangle soup [S2, unofficial].
*   **`mod2`**: Uses mod2 intersection reduction before subgrid construction [S2, unofficial].
*   **`greedy`**: Uses greedy construction (fan triangulation from boundary curves) [S2, unofficial].
*   **`cgal`**: Uses exact CGAL (EPECK) intersection queries. Requires build with `-DSUBGRID_WITH_CGAL=ON`. Only applies to mesh input [S2, unofficial].
*   **`USE_ROBUST_QUERIES`**: Static constexpr bool. Default `false`. Enables FCPW watertight test + iterative all-hits ray intersection. Disabled by default due to ~2x slowdown [S2, unofficial].
*   **`scoop_bulge`**: Controls bulging parameter for scoop curves. Default `1e-3`. Affects intersection-free property [S2, unofficial].
*   **`scoop_mid_vertices`**: Controls mid-vertex placement for scoop curves [S2, unofficial].
*   **`resolution`**: Integer $n$ for $n^3$ grid. Typical values: 32, 64, 128 [S2, unofficial].

### Edge Cases and Pitfalls

*   **Non-Even Tets:** If the even-sum condition is not satisfied (`non_even_tets > 0`), the output mesh may not be orientable and may contain open holes. This typically indicates a non-watertight input or a grid edge-intersection issue [S2, unofficial].
*   **Input Validation:** The program requires exactly one input type (`--input mesh`, `--inputSDF`, or `--npz`). Passing a `.npz` file to `--input` triggers an error [S2, unofficial].
*   **CGAL Usage:** The `--cgal` flag applies only to mesh input (`-i`). Using it with SDF or NPZ inputs triggers an error [S2, unofficial].
*   **Self-Intersections:** The output is mathematically guaranteed to be free of self-intersections. Numerical tests may flag near-overlapping triangles as false positives [S2, unofficial].

## Subtopics

### [Marching Tetrahedrons](3720-marching-tetrahedrons.md)
Marching Tetrahedrons is an isosurface extraction algorithm that decomposes each voxel of a 3D grid into tetrahedra to determine the intersection of a scalar field with a target isovalue. Unlike its sibling [Marching Cubes](4328-naive-isosurface-extraction.md), which suffers from topological ambiguities in specific configurations, this method guarantees a manifold, intersection-free surface by resolving these ambiguities through tetrahedral decomposition. It produces a mesh that matches voxel data more accurately than Marching Cubes, at the cost of a higher vertex count. Choose this when topological correctness is critical and vertex count is not a primary constraint.

### [Optimal Extraction via Partition Index](4355-optimal-extraction-via-partition-index.md)
Optimal Extraction via Partition Index is a method for extracting isosurfaces from a tetrahedral grid that guarantees visiting only the tetrahedra actually intersected by the level set, thereby achieving optimal runtime complexity for the extraction phase. It distinguishes itself from standard marching tetrahedra by pre-processing the mesh into partitions separated by saddle contours, allowing the algorithm to jump directly to connected components of the isosurface via a partition index. Choose this for large sparse datasets where most of the grid is empty.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of tetrahedra in the grid, $O(n^3)$ for an $n^3$ grid. The local reconstruction per tetrahedron is constant time.
*   **Memory Complexity:** Linear with respect to the number of tetrahedra and the resulting mesh size. Storage for edge intersections and vertex merging structures is required.
*   **Measured Performance:** Benchmarking shows that enabling robust queries (`USE_ROBUST_QUERIES`) results in a ~2x slowdown with negligible effect on robustness [S2, unofficial].
*   **Strengths:** Guarantees intersection-free, manifold surfaces. Resolves topological ambiguities inherent in Marching Cubes. Supports multiple input types (mesh, SDF, precomputed intersections).
*   **Limitations:** Higher vertex count than Marching Cubes. Sensitive to non-watertight inputs (results in non-even tets). Numerical precision issues can arise at low resolutions or with degenerate inputs.
*   **When to Use:** When topological correctness and intersection-free guarantees are required. Suitable for medical imaging, scientific visualization, and CAD data.
*   **When Not to Use:** When real-time performance is critical and approximate surfaces are acceptable. Not ideal for highly sparse datasets where partition-based methods are more efficient.

## Background
Subgrid Marching Tetrahedra was introduced by H. Baktash et al. as a method for extracting intersection-free isosurfaces. The algorithm addresses the topological ambiguities of Marching Cubes by using a tetrahedral decomposition and a consistent local reconstruction policy. The method has been implemented in both C and Python, with support for various input types and reconstruction policies. The official implementation is available on GitHub [S2, unofficial], and an unofficial reference implementation is also available [S1, unofficial].

## Sources
[S1] DocSavage/SubgridMarchingTetrahedraCode, https://github.com/DocSavage/SubgridMarchingTetrahedraCode, fetched 2026-10-26 (unofficial)
[S2] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
