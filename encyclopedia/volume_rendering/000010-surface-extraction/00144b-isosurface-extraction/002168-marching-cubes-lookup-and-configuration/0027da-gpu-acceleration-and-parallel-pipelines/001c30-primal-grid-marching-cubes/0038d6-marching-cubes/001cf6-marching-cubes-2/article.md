# Marching Cubes 2


## Overview
Marching Cubes 2 is a variant of the Marching Cubes algorithm that modifies the standard lookup table and configuration logic to resolve topological ambiguities, specifically face-saddle ambiguities in the trilinear interpolant [S3]. It increases the number of topological configurations from 15 to 24 to effectively avoid the generation of voids in the extracted mesh [S4, abstract only, unofficial].

## How it works
**Inputs:**
*   A 3D scalar field (grid) of size $N_x \times N_y \times N_z$ [S1, unofficial][S4, abstract only, unofficial].
*   An isovalue (isolevel) $I$ [S1, unofficial][S4, abstract only, unofficial].
*   The grid data array [S1, unofficial].

**Outputs:**
*   A list of vertices and polygons representing the isosurface [S1, unofficial].

**Data Structures:**
*   **Grid:** A 3D array of scalar values.
*   **Lookup Table:** An extended table with 24 topological configurations (increased from the standard 15) [S4, abstract only, unofficial].
*   **Thread Classification:** The 24 configurations are divided into three types for parallel processing [S4, abstract only, unofficial].

**Steps:**
1.  **Preparation:** Validate the 3D grid dimensions, extract the shape, compute bounds $N_x$, $N_y$, and $N_z$, and initialize output vectors for vertices and polygons [S1, unofficial].
2.  **Configuration Classification:** Divide the 24 topological configurations into three classes, where each class corresponds to a specific thread for multi-thread parallel processing [S4, abstract only, unofficial].
3.  **Ambiguity Resolution:** For each cube, determine the case index. If the case involves a face-saddle ambiguity, resolve the topology using the extended lookup table or an asymptotic decider [S3].
4.  **Vertex Generation:** Use midpoint selection instead of linear interpolation to calculate vertex positions on intersected edges, improving reconstruction speed [S4, abstract only, unofficial].
5.  **Parallel Processing:** Divide grid processing among threads based on the configuration classes. Use a critical region to realize multi-thread synchronization when writing to shared output structures [S4, abstract only, unofficial]. Design a protocol mapping table to encapsulate processing functions and trigger them via function pointers [S4, abstract only, unofficial].
6.  **Connectivity Table Generation:** Iterate through all 256 cases of the standard Marching Cubes `tri_table`. For each case, map the edge indices from `tri_table` to vertex indices using a vertex lookup array. If an edge index is not found or is 0, assign 0 to the connectivity table entry; otherwise assign the found vertex index [S5, unofficial].
7.  **Output Assembly:** Collect the generated vertices and polygons into the final output structures [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes2(grid, isovalue, numx, numy, numz):
    # 1. Preparation: validate 3D grid, extract shape, compute bounds/numx/numy/numz, init vectors [S1, unofficial]
    assert len(grid.shape) == 3
    vertices = []
    polygons = []
    edge_to_vertex_map = {}

    # 2. Configuration Classification: divide the 24 topological configurations into three types [S4, abstract only, unofficial]
    config_types = {0: list(range(15)), 1: list(range(15, 24)), 2: list(range(24, 32))}

    # 3. Multi-thread parallel processing [S4, abstract only, unofficial]
    import threading
    threads = []
    for type_id in range(3):
        t = threading.Thread(target=process_config_type, args=(grid, isovalue, numx, numy, numz, type_id, config_types[type_id], vertices, polygons, edge_to_vertex_map))
        threads.append(t)
        t.start()
    for t in threads:
        t.join()

    return vertices, polygons

def process_config_type(grid, isovalue, numx, numy, numz, config_type, valid_cases, vertices, polygons, edge_to_vertex_map):
    # Iterate over all voxels
    for x in range(numx - 1):
        for y in range(numy - 1):
            for z in range(numz - 1):
                # Get cube corner values
                cube_vals = [grid[x, y, z], grid[x+1, y, z], grid[x+1, y+1, z], grid[x, y+1, z],
                             grid[x, y, z+1], grid[x+1, y, z+1], grid[x+1, y+1, z+1], grid[x, y+1, z+1]]
                # Determine case index
                case_index = 0
                for i, v in enumerate(cube_vals):
                    if v >= isovalue:
                        case_index |= (1 << i)

                if case_index not in valid_cases:
                    continue

                # Resolve face-saddle ambiguity if necessary [S3]
                # (Handled by extended lookup table or decider)

                # Vertex Generation: midpoint selection instead of linear interpolation [S4, abstract only, unofficial]
                edge_intersections = []
                edge_coords = [(0,0,0), (1,0,0), (1,1,0), (0,1,0), (0,0,1), (1,0,1), (1,1,1), (0,1,1)]
                edges = [(0,1), (1,2), (2,3), (3,0), (4,5), (5,6), (6,7), (7,4), (0,4), (1,5), (2,6), (3,7)]
                for e_idx in range(12):
                    v1, v2 = edges[e_idx]
                    if (cube_vals[v1] >= isovalue) != (cube_vals[v2] >= isovalue):
                        mx, my, mz = (edge_coords[v1][0] + edge_coords[v2][0])/2.0, \
                                     (edge_coords[v1][1] + edge_coords[v2][1])/2.0, \
                                     (edge_coords[v1][2] + edge_coords[v2][2])/2.0
                        edge_intersections.append((mx, my, mz))

                # Connectivity Table Generation [S5, unofficial]
                # Map edge indices from tri_table to vertex indices
                tri_table = [...] # Standard 256-case lookup table
                for tri in tri_table[case_index]:
                    if tri == -1: break
                    v_idx = edge_to_vertex_map.get(tri, 0)
                    # Build triangle from mapped vertices
                    pass

                # Synchronization for critical region [S4, abstract only, unofficial]
                with threading.Lock():
                    polygons.append(triangles)
```

**Parameters:**
*   **Isovalue:** The scalar threshold for extraction.
*   **Grid Resolution:** $N_x, N_y, N_z$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Voids:** The standard MC algorithm (15 configurations) can generate voids. This is handled by increasing configurations to 24 [S4, abstract only, unofficial].
*   **Face-Saddle Ambiguities:** The trilinear interpolant has face-saddle ambiguities that require resolution (e.g., via Asymptotic Decider) [S3].
*   **Thread Synchronization:** Multi-threading requires careful synchronization using critical regions to avoid race conditions when writing to shared output structures [S4, abstract only, unofficial].

## Subtopics

### [Extended Marching Cubes with No Degenerate Triangles](2186-extended-marching-cubes-with-no-degenerate-triangles.md)
A variant that uses an extended lookup table to avoid degenerate triangles and ensures separation of positive/negative regions.

### [Extended Marching Cubes Lookup Table](2187-extended-marching-cubes-lookup-table.md)
A lookup table distinguishing scalar values equal to the isovalue from those greater, avoiding degenerate triangles.

### [Contouring Discrete Indicator Functions](4087-contouring-discrete-indicator-functions.md)
A method for DIFs that replaces linear interpolation with area-matching solutions on a dual grid.

### [2D Reduction for 3D DIF Contouring](4301-2d-reduction-for-3d-dif-contouring.md)
Uses 2D intersection formulas for 3D grid edges for more accurate approximation of linear functions.

### [Marching Cubes with Topological Correction (Case 6c)](4349-marching-cubes-with-topological-correction-case-6c.md)
Resolves topological inconsistencies (holes) by using alternate facetization for Case 6 adjacent to Case 3.

### CUDA Isosurface Extraction with Overlapped Execution
An optimization for CUDA-based isosurface extraction that overlaps data transfers and kernel executions to hide latency [S2]. It executes three overlapping pairs of operations: 1) calculation of vertices and transfer of triangles from the previous slice pair; 2) calculation of triangles and transfer of vertices; 3) compaction of triangles and transfer of the next slice [S2]. Constant memory is used for the lookup table and isovalue [S2]. Device initialization takes ~4 seconds, negligible for large datasets but significant for small ones (<100 MB) [S2]. For a 2048x2048x6936 dataset (23 GB), 16 GTX480 devices achieved speedups of 752.3x (isovalue 1000) and 1489.7x (isovalue 4000) [S2].

### Finite State Machine for Interpolant Modelling
A representation of isosurface evolution by sweeping from high to low isovalues, where states are cases and transitions occur when a vertex changes from white (lower than $h$) to black (higher than $h$) [S3]. The FSM is constructed by defining states as cases and adding transitions from each state to each other case with one extra black vertex [S3]. Each cube starts with case 0. For example, case 2 (two edge-adjacent black vertices) cannot sweep to case 7 (no edge-adjacent black vertices) because topology forbids losing edge adjacency [S3]. This is used to model watertight Marching Cubes cases and prove completeness of Nielson's cases [S3].

## Operating characteristics
*   **Speed:** The improved MC algorithm (with 24 configurations and midpoint selection) is approximately 30% faster than the standard MC algorithm and 40% faster than the Masala algorithm [S4, abstract only, unofficial].
*   **Parallelism:** Uses multi-thread parallel processing with 3 thread types for 24 configurations, synchronized via critical regions [S4, abstract only, unofficial].
*   **Memory:** Uses constant memory for lookup tables and isovalue in GPU implementations [S2].
*   **Accuracy:** Increases topological configurations to 24 to avoid voids [S4, abstract only, unofficial]. Resolves face-saddle ambiguities using the Asymptotic Decider [S3].
*   **Use Cases:** Medical image 3D reconstruction, where accuracy and speed are critical [S4, abstract only, unofficial].

## Background
Marching Cubes 2 was introduced by Nielson & Hamann in 1991 in the paper "The asymptotic decider: Resolving the ambiguity in marching cubes" [S3]. It was developed to address the face-saddle ambiguities present in the original trilinear interpolant used in the standard Marching Cubes algorithm [S3]. The variant with 24 configurations and midpoint selection was later proposed in a 2020 paper for improved 3D reconstruction speed and void avoidance [S4, abstract only, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] A parallel isosurface extraction component for visualization pipelines executing on GPU clusters, https://doi.org/10.1016/j.cam.2014.05.019, fetched 2026-10-27
[S3] (No) More Marching Cubes, https://doi.org/10.2312/vg/vg07/081-090, fetched 2026-10-26
[S4] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
[S5] JuliaGeometry/Meshing.jl, https://github.com/JuliaGeometry/Meshing.jl, fetched 2026-10-27 (unofficial)
