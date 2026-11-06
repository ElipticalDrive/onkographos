# Subgrid Marching Tetrahedra

Also called: Subgrid MT, single-tet reconstruction

## Overview
Subgrid Marching Tetrahedra (Subgrid MT) is an isosurface extraction method that reconstructs an intersection-free triangulation by decomposing a regular grid into tetrahedra and solving for the surface within each tetrahedron individually [S1, S2, unofficial]. Unlike its sibling [Marching Cubes](4328-naive-isosurface-extraction.md), which suffers from topological ambiguities in specific configurations, Subgrid MT guarantees a manifold, intersection-free surface by resolving these ambiguities through tetrahedral decomposition [S1, S4, unofficial]. It produces a mesh that matches voxel data more accurately than Marching Cubes, at the cost of a higher vertex count [S2, unofficial].

## How it works
Subgrid MT operates in two phases: global grid traversal to determine edge intersections, and local single-tetrahedron reconstruction to generate triangles. The method accepts three input types: a triangle mesh remeshed on a grid, an analytical Signed Distance Function (SDF), or precomputed edge intersections in `.npz` format [S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   **Mesh**: A triangle mesh file (OBJ, PLY, OFF). The input is automatically welded, triangulated, centered, and scaled to fit within 98% of the unit cube $[-1, 1]^3$. A rigid translation with a fixed seed offset (magnitude 0.01) is applied to break axis-alignment [S2, unofficial].
    *   **SDF**: A named analytical SDF (e.g., `Sphere`, `Torus`) [S2, unofficial].
    *   **Explicit Intersections**: A `.npz` file containing an explicit tetrahedral mesh and precomputed edge intersection counts/t-values [S2, unofficial].
*   **Outputs**:
    *   **Primal**: A triangle soup or manifold mesh.
    *   **Dual**: A dual polygon mesh generated via Quadric Error Function (QEF) solve [S2, unofficial].

### Data Structures

#### Single-Tetrahedron Reconstruction (`sgmt_result`)
The core reconstruction routine returns a struct containing:
*   `triangles`: `int32_t` array, 3 indices per triangle.
*   `num_triangles`: Count of triangles.
*   `num_edge_points`: Count of vertices located on edges.
*   `num_steiner`: Count of Steiner points (internal vertices).
*   `steiner_bary`: Array of 4 barycentric coordinates per Steiner point [S1, unofficial].

#### Full Mesh (`sgmt_mesh`)
The final mesh structure contains:
*   `positions`: Array of 3 coordinates per vertex.
*   `triangles`: Array of 3 indices per triangle.
*   `num_vertices`: Count of vertices.
*   `num_triangles`: Count of triangles.
*   `ok`: Boolean success flag [S1, unofficial].

#### Python Result Structures
*   `PrimalResult`: Contains `vertices` (N, 3) `float64` array and `faces` (list of index arrays).
*   `DualResult`: Contains dual polygon mesh data [S2, unofficial].

### Algorithm Steps

1.  **Grid Decomposition**: Decompose the regular Cartesian grid into tetrahedra. A standard $n \times n \times n$ grid of cubes is split into $5n^3$ tetrahedra [S2, unofficial].
2.  **Edge Intersection Detection**: For each tetrahedron, determine how many times the isosurface crosses each of its six edges.
    *   For **Mesh/SDF inputs**: Ray casting or analytical evaluation determines intersections. If using mesh input, CGAL (EPECK) can be used for exact arithmetic [S2, unofficial].
    *   For **Explicit inputs**: Read precomputed counts from the `.npz` file.
    *   The edge coordinates are stored in paper order: $(e_{01}, e_{02}, e_{03}, e_{23}, e_{13}, e_{12})$ [S1, unofficial].
3.  **Validation**: Validate that exactly six non-negative edge coordinates are provided for each tetrahedron. If validation fails, the process aborts [S1, unofficial].
4.  **Local Reconstruction**: Call the reconstruction routine for each tetrahedron with its edge coordinates and the simplicial embedding flag.
    *   The routine reconstructs an intersection-free triangulation interpolating the surface crossings [S1, unofficial].
    *   Handle failure by checking the `ok` flag or result validity [S1, unofficial].
5.  **Vertex Merging**: Merge vertices across adjacent tetrahedra to form a coherent mesh.
    *   **Combinatorial Merge** (default): Merges vertices based on topological signatures.
    *   **Numerical Merge**: Switched on by passing `mergeEPS >= 0`. Merges vertices within a positional epsilon.
    *   **No Merge**: Disables all vertex merging, outputting a raw triangle soup (`noMerge` flag) [S2, unofficial].
6.  **Output Generation**: Iterate over the result triangles array in strides of three, outputting the three vertex indices for each triangle. If Steiner points are present, output their barycentric coordinates and global indices [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subgrid_marching_tetrahedra(input_type, input_data, resolution=64, merge_eps=-1, simplicial=True):
    """
    Reconstructs an intersection-free isosurface from a grid.
    
    Args:
        input_type: 'mesh', 'sdf', or 'npz'
        input_data: Path to mesh/SDF name or .npz file
        resolution: Grid resolution n (creates n^3 cubes)
        merge_eps: Positional merge epsilon. < 0 for combinatorial merge.
        simplicial: Use simplicial embedding (default True)
    
    Returns:
        sgmt_mesh object containing positions and triangles
    """
    
    # 1. Initialize Grid and Tetrahedralization
    # Create an n x n x n grid. Decompose each cube into 5 tetrahedra.
    # Total tets = 5 * resolution^3
    grid = create_grid(resolution)
    tetrahedra = decompose_grid_to_tets(grid)
    
    # 2. Compute Edge Intersections
    # For each tetrahedron, determine intersection counts for its 6 edges.
    edge_counts = {} # Map: tet_index -> [e01, e02, e03, e23, e13, e12]
    
    if input_type == 'mesh':
        # Preprocess mesh: weld, triangulate, scale to [-1,1]^3, apply seed offset
        mesh = preprocess_mesh(input_data)
        for tet in tetrahedra:
            # Ray cast against mesh to find edge intersections
            counts = ray_cast_edge_intersections(tet, mesh)
            edge_counts[tet.id] = counts
            
    elif input_type == 'sdf':
        sdf_func = get_sdf_function(input_data)
        for tet in tetrahedra:
            # Evaluate SDF at vertices or ray cast to find edge crossings
            counts = sdf_edge_intersections(tet, sdf_func)
            edge_counts[tet.id] = counts
            
    elif input_type == 'npz':
        # Load precomputed intersections
        data = load_npz(input_data)
        edge_counts = data.edge_intersections
        
    # 3. Local Reconstruction and Merging
    all_vertices = []
    all_triangles = []
    vertex_map = {} # Map edge/vertex signature to global index
    
    for tet in tetrahedra:
        counts = edge_counts[tet.id]
        
        # Validate inputs
        if len(counts) != 6 or any(c < 0 for c in counts):
            raise ValueError("Invalid edge coordinates")
            
        # Reconstruct local surface
        # Returns local triangles and any Steiner points
        local_result = sgmt_reconstruct_tet(counts, simplicial)
        
        # Merge vertices
        # Use combinatorial or numerical merge based on merge_eps
        merged_triangles = merge_local_triangles(
            local_result.triangles, 
            local_result.vertices, 
            vertex_map, 
            merge_eps
        )
        
        all_triangles.extend(merged_triangles)
        
    # 4. Finalize Mesh
    mesh = sgmt_mesh(
        positions=all_vertices,
        triangles=all_triangles,
        num_vertices=len(all_vertices),
        num_triangles=len(all_triangles),
        ok=True
    )
    
    return mesh

def sgmt_reconstruct_tet(edge_coords, simplicial):
    """
    Reconstructs a single tetrahedron's surface from its six edge coordinates.
    
    Args:
        edge_coords: Array of 6 non-negative integers [e01, e02, e03, e23, e13, e12]
        simplicial: Boolean flag for simplicial embedding
        
    Returns:
        sgmt_result struct with triangles, edge points, and Steiner points
    """
    # Define vertices of a regular (equilateral) tetrahedron if not provided
    # v0 = (0, sqrt(6)/4, 0)
    # v1 = (-0.5, -sqrt(6)/12, -sqrt(3)/6)
    # ... (other vertices)
    
    # Determine curve types (open, scoop, normal) based on edge counts
    # Apply construction policy (primal, dual, or greedy)
    
    # Construct local TriangleSoup
    # Handle even-sum condition for watertightness
    # If even-sum condition is not satisfied, output may have holes
    
    return sgmt_result(
        triangles=[], # Populated by construction policy
        num_triangles=0,
        num_edge_points=0,
        num_steiner=0,
        steiner_bary=[]
    )
```

### Parameters
*   **`edge_coords[6]`**: Array of six non-negative integers representing intersection counts per edge in paper order: $(e_{01}, e_{02}, e_{03}, e_{23}, e_{13}, e_{12})$ [S1, unofficial].
*   **`simplicial`**: Boolean flag controlling whether to use the simplicial embedding. Default value is `1` (on) [S1, unofficial].
*   **`mergeEPS`**: A positional merge epsilon. Passing a value $\ge 0$ switches from the default combinatorial merge to numerical merge. Default is `-1` (combinatorial) [S2, unofficial].
*   **`noMerge`**: Disables all vertex merging, outputting a raw triangle soup [S2, unofficial].
*   **`mod2`**: Uses mod2 intersection reduction before subgrid construction [S2, unofficial].
*   **`greedy`**: Uses greedy construction, specifically fan triangulation from boundary curves [S2, unofficial].
*   **`cgal`**: Uses exact CGAL (EPECK) intersection queries for mesh input. Requires build with `-DSUBGRID_WITH_CGAL=ON` [S2, unofficial].
*   **`USE_ROBUST_QUERIES`**: A static constexpr bool set to `false` in `main_subgrid.cpp`. It enables FCPW watertight test + iterative all-hits ray intersection, but is disabled by default because benchmarking showed it ~2x slower with negligible effect on robustness [S2, unofficial].
*   **`scoop_bulge`**: Controls bulging parameter for scoop curves. Default `1e-3`. A small enough value guarantees the intersection-free property [S2, unofficial].
*   **`scoop_mid_vertices`**: Controls mid-vertex placement for scoop curves [S2, unofficial].

### Edge Cases and Pitfalls
*   **Non-Even Tets**: Each active tetrahedron should see the isosurface cross its faces an even number of times (the "even-sum" condition). If this condition is not satisfied (`non_even_tets > 0`), the output mesh might not be orientable and may contain open holes. This typically indicates a non-watertight input or a grid edge-intersection issue [S2, unofficial].
*   **Input Validation**: The program requires exactly one input type (`--input mesh`, `--inputSDF`, or `--npz`). Passing a `.npz` file to `--input` triggers an error [S2, unofficial].
*   **CGAL Usage**: The `--cgal` flag applies only to mesh input (`-i`). Using it with SDF or NPZ inputs triggers an error [S2, unofficial].
*   **Self-Intersections**: The output is mathematically guaranteed to be free of self-intersections. Numerical tests may flag near-overlapping triangles as false positives [S2, unofficial].

## Subtopics

### [Marching Tetrahedrons](3720-marching-tetrahedrons.md)
Marching Tetrahedrons is an isosurface extraction algorithm that decomposes each voxel of a 3D grid into tetrahedra to determine the intersection of a scalar field with a target isovalue. Unlike its sibling [Marching Cubes](4328-naive-isosurface-extraction.md), which suffers from topological ambiguities in specific configurations, this method guarantees a manifold, intersection-free surface by resolving these ambiguities through tetrahedral decomposition. It produces a mesh that matches voxel data more accurately than Marching Cubes, at the cost of a higher vertex count. Choose this when topological correctness is critical and vertex count is not a primary constraint.

### [Optimal Extraction via Partition Index](4355-optimal-extraction-via-partition-index.md)
Optimal Extraction via Partition Index is a method for extracting isosurfaces from a tetrahedral grid that guarantees visiting only the tetrahedra actually intersected by the level set, thereby achieving optimal runtime complexity for the extraction phase. It distinguishes itself from standard marching tetrahedra by pre-processing the mesh into partitions separated by saddle contours, allowing the algorithm to jump directly to connected components of the isosurface via a partition index. The method builds on [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) to guarantee intersection-free output. Choose this for large sparse datasets where traversal efficiency is paramount.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of tetrahedra in the grid. The complexity is dominated by the edge intersection detection phase for mesh/SDF inputs.
*   **Memory Complexity**: Linear with respect to the number of tetrahedra and the resulting mesh size.
*   **Measured Performance**: Benchmarking shows that enabling robust queries (`USE_ROBUST_QUERIES`) is ~2x slower with negligible effect on robustness [S2, unofficial].
*   **Strengths**: Guarantees intersection-free, manifold output. Resolves topological ambiguities present in Marching Cubes. Supports both primal and dual extraction modes.
*   **Limitations**: Higher vertex count than Marching Cubes. Requires tetrahedral decomposition. Sensitive to non-watertight inputs (results in non-even tets).
*   **When to Use**: When topological correctness and intersection-free output are required, and the input data can be represented on a regular grid.
*   **When Not to Use**: When vertex count minimization is critical, or when the input data is not suitable for regular grid decomposition.

## Background
Subgrid Marching Tetrahedra was introduced by H. Baktash et al. as a method for extracting intersection-free isosurfaces. The method builds on the principles of Marching Tetrahedrons but introduces specific reconstruction policies to ensure intersection-free results. The implementation is available in C++ and Python, with support for various input types including meshes, SDFs, and precomputed intersections [S1, S2, unofficial].

## Sources
[S1] DocSavage/SubgridMarchingTetrahedraCode, https://github.com/DocSavage/SubgridMarchingTetrahedraCode, fetched 2026-10-26 (unofficial)
[S2] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
