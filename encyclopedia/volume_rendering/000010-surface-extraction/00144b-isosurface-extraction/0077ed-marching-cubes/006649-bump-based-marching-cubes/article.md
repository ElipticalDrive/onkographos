# Bump-based Marching Cubes

Also called: Bump MC, Bump-backed MarchingCubes

## Overview
Bump-based Marching Cubes is a policy-based isosurface extraction method that leverages the Bump library to support both structured (Cartesian) and unstructured (quad/hex) meshes, unlike the native implementation which only supports structured grids [S1, unofficial]. It generates a polygonal mesh and subsequently triangulates it, performing vertex welding to ensure manifold connectivity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on a grid (structured or unstructured quad/hex).
*   **Output:** A triangle mesh representing the isosurface.

**Data Structures:**
The implementation relies on Bump's data structures for grid traversal and field intersection. Key internal arrays include:
*   `crossing_flags`: Boolean or bitmask indicating which cell edges intersect the isosurface.
*   `old-to-new-map`: Array mapping original cell indices to compacted indices after filtering.
*   `node-map`: Maps grid nodes to unique vertex indices in the output mesh.
*   `polygon_mesh`: Intermediate structure holding vertices and polygonal faces before triangulation.

**Numbered Steps:**

1.  **Crossing Selection Prefilter:**
    1.  **Crossing Classification:** Iterate over cells. For each cell, determine which edges cross the isosurface based on vertex scalar values. Store these as crossing flags [S1, unofficial].
    2.  **Crossing Scan:** Perform a parallel scan (prefix sum) on the crossing flags to determine the number of crossings per cell and their offsets. Optimizations include reusing crossing flags for the scan so an explicit offsets array is not needed [S1, unofficial].
    3.  **Crossing Compaction:** Compact the list of cells containing crossings. This involves scanning directly into the `old-to-new-map` array and avoiding initialization of arrays that will be immediately overwritten [S1, unofficial].

2.  **CutField Phase:**
    1.  **Classification and Sizing:** For each cell with crossings, classify the topological case (using a bitmask to check vertex labels/cases) and determine the size of the polygon to be generated. Empty cells are labeled and skipped [S1, unofficial].
    2.  **Weld Pipeline:** Identify vertices that should be merged (welded) to ensure topological correctness. This step is performed by the Bump algorithm but not the native algorithm, contributing to higher CPU time [S1, unofficial].
    3.  **Polygon Mesh Construction:** Generate the polygonal faces for the isosurface within each cell. The `node-map` is reused to avoid large temporary allocations [S1, unofficial].
    4.  **Other CutField Work:** Additional processing required by the Bump library's field intersection logic [S1, unofficial].

3.  **Polygon-to-Triangle Adaptation:**
    1.  Convert the generated polygonal mesh into a triangle mesh. This is an extra step compared to the native implementation which generates triangles directly, serving as a source of overhead [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bump_based_marching_cubes(grid, scalar_field, isovalue):
    # 1. Crossing Selection Prefilter
    # Classify crossings for all cells
    crossing_flags = classify_crossings(grid, scalar_field, isovalue)
    
    # Scan to find offsets; reuse flags to avoid extra array
    # inclusive_scan returns cumulative counts
    scan_offsets = inclusive_scan(crossing_flags)
    
    # Compact cells with crossings
    # old_to_new_map maps original index to new compacted index
    old_to_new_map = compact_cells(crossing_flags, scan_offsets)
    
    # 2. CutField Phase
    # Initialize node map for welding
    node_map = initialize_node_map(grid.num_nodes)
    
    # Process only compacted cells
    for cell_idx in range(num_compacted_cells):
        original_cell_idx = old_to_new_map[cell_idx]
        
        # Classify case using bitmask
        case = get_cell_case(grid, scalar_field, original_cell_idx, isovalue)
        
        if case == 0:
            continue # Empty cell
            
        # Weld vertices to ensure manifold connectivity
        welded_vertices = weld_vertices(grid, scalar_field, original_cell_idx, isovalue, node_map)
        
        # Generate polygonal faces
        polygons = generate_polygons(welded_vertices, case)
        
        # Add polygons to mesh
        add_polygons_to_mesh(polygons)
        
    # 3. Polygon-to-Triangle Adaptation
    # Triangulate the polygonal mesh
    triangle_mesh = triangulate_polygon_mesh()
    
    return triangle_mesh
```

**Parameters:**
*   **Memory Pool:** The implementation uses an Umpire QuickPool with an initial allocation of 20GB [S1, unofficial].
*   **Execution Policy:** Supports CPU, OpenMP, CUDA, and HIP execution spaces [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **HIP Buffer Allocation:** On HIP execution, timings may include buffer allocation and initialization overhead, which is not present in CUDA timings due to `nvcc` optimization. This can lead to misleading performance comparisons if not accounted for [S1, unofficial].
*   **MSVC Link Error:** On MSVC with shared library configuration and `CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS=ON`, the large number of exported symbols from MC objects can cause a link error (LNK1189). The fix is to make the MC object a private static library [S1, unofficial].
*   **Unstructured Hex Mesh Output:** When saving MC output for unstructured hex meshes on CUDA, the data resides on the device. It must be copied to the host before saving to avoid errors [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The method performs more work than the native Marching Cubes implementation on structured grids due to vertex welding and the extra triangulation step [S1, unofficial].

**Measured Performance:**
Benchmarked on a $960^3$ grid over $[0,2\pi]^3$ using a gyroid scalar field at scale $(1,1,1)$, generating 8,933,625 triangles [S1, unofficial].

| Method | Seq (s) | HIP (s) | Seq (s) | CUDA (ms) |
| :--- | :--- | :--- | :--- | :--- |
| **Native MC** | 9.09 | 5.26 | 8.63 | 146.59 |
| **Bump MC** | 17.98 | 821.35 | 13.03 | 166.65 |

*Note: Seq entries are inclusive (exclusive). GPU entries are inclusive (exclusive; %GPU). The "rzadams" and "rzvector" labels refer to specific hardware configurations in the source [S1, unofficial].*

**Scaling and Limitations:**
*   Bump MC is around the same time as Native MC on CUDA but significantly slower on HIP due to buffer allocation overhead [S1, unofficial].
*   Bump CPU times take about twice as long as their native counterpart, largely due to welding [S1, unofficial].
*   The method is slower on structured grids than the native implementation but provides the advantage of supporting unstructured meshes [S1, unofficial].

**When to Use:**
Use Bump-based Marching Cubes when working with unstructured quad or hex meshes, or when the flexibility of the Bump library's execution policies (HIP/CUDA/OpenMP) is required. Avoid it if maximum performance on structured grids is the primary concern, as the native implementation is faster [S1, unofficial].

## Background
Bump-based Marching Cubes was introduced as part of the Axom Quest library to extend Marching Cubes functionality beyond structured grids. The original implementation only supported structured (Cartesian) grids, but user requests necessitated support for unstructured quad and hex meshes [S1, unofficial]. The implementation is policy-based, allowing execution on CPU, OpenMP, CUDA, and HIP backends [S1, unofficial].

## Sources
[S1] Adds a bump-based Marching Cubes option, https://github.com/llnl/axom/pull/1950, fetched 2026-10-26 (unofficial)
