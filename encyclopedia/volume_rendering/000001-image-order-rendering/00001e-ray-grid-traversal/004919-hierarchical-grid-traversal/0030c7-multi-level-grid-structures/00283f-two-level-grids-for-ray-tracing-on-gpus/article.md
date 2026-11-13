# Two-Level Grids for Ray Tracing on GPUs

Also called: Nested Grids, Two-Level Grid, Two-Level Grids for Ray Tracing, Two-Level Nested Grid

## Overview
Two-Level Grids for Ray Tracing on GPUs is a fixed-depth hierarchical acceleration structure designed to handle non-uniform primitive distributions more robustly than uniform grids, specifically addressing the "teapot in a stadium problem" where sparse regions fail to eliminate intersection candidates [S1, S2, unofficial]. It consists of a sparse top-level grid where each occupied cell points to a finer, arbitrary-resolution sub-grid (leaf grid) [S1, unofficial]. Unlike its siblings, it uses a single global sort to construct all leaf-level references in parallel, optimizing for GPU memory constraints and build speed, while traversal reuses decision variables between levels to maintain SIMD coherence [S1, S2, unofficial].

## How it works
**Inputs:** Triangle mesh data ($t$), scene bounding box ($b$), top-level grid resolution ($r$).
**Outputs:** Reference Array, Leaf Array, Top-Level Cells Array.

### Data Structures
The structure is defined by three arrays [S1, S2, unofficial]:

1.  **Reference Array:** A flat array storing pairs of `(leaf_cell_index, primitive_id)`. Each primitive is referenced once for every leaf cell it overlaps. References for the same leaf cell are stored contiguously.
2.  **Leaf Array:** Stores the start and end indices into the Reference Array for each leaf cell. If a leaf cell is empty, the interval is `[i, i)`.
3.  **Top-Level Cells Array:** Stores metadata for each top-level cell. Each entry consists of two 32-bit integers [S1, S2, unofficial]:
    *   **First Leaf Index:** A 32-bit integer indicating the index of the first leaf cell in the Leaf Array.
    *   **Cell Info:** A 32-bit integer where:
        *   Lower 24 bits store the cell resolution (limiting resolution to $256^3$).
        *   Upper 8 bits are flags (e.g., 'is empty', 'has children').

### Construction Algorithm
The construction is data-parallel and sort-based [S1, S2, unofficial]. It builds the top-level grid first, then computes leaf locations, counts references, and performs a single sort to organize primitive references.

**Parameters:**
*   $G$: Grid dimension for accumulation arrays. In the source, $G=(r_y, r_z)$ for leaf cell counting and $G=128$ for reference counting [S2, unofficial].
*   $B$: Block size. $B=r_x$ for leaf counting, $B=256$ for reference counting [S2, unofficial].

**Steps:**

1.  **Compute Bounds and Resolution:**
    *   Compute the bounding box of all primitives: $b \leftarrow \text{COMPUTE BOUNDS}(t)$ [S2, unofficial].
    *   Compute the top-level grid resolution: $r \leftarrow \text{COMPUTE TOP LEVEL RESOLUTION}(b)$ [S2, unofficial].
    *   Upload triangle data to GPU: $t \leftarrow \text{UPLOAD TRIANGLES}(t)$ [S2, unofficial].

2.  **Build Top-Level Grid:**
    *   Build a temporary uniform grid for initialization: $\text{data} \leftarrow \text{BUILD UNIFORM GRID}(t, b, r)$ [S1, S2, unofficial].
    *   Get sorted top-level pairs (cell index, primitive ID): $\text{Atl\_p} \leftarrow \text{GET SORTED TOP-LEVEL PAIRS}(\text{data})$ [S1, S2, unofficial].
    *   Get the number of top-level references: $n \leftarrow \text{GET NUMBER OF TOP-LEVEL REFERENCES}(\text{data})$ [S1, S2, unofficial].
    *   Get top-level cell ranges: $\text{tlc} \leftarrow \text{GET TOP-LEVEL CELL RANGES}(\text{data})$ [S1, S2, unofficial].

3.  **Compute Leaf Cell Locations:**
    *   Initialize an accumulator array $\text{Acc}$ of size $r_x \cdot r_y \cdot r_z + 1$ with zeros [S1, S2, unofficial].
    *   Count leaf cells per top-level cell: $\text{Acc} \leftarrow \text{COUNT LEAF CELLS}(b, r, \text{tlc})$ [S1, S2, unofficial].
    *   Perform exclusive scan on $\text{Acc}$: $\text{Acc} \leftarrow \text{EXCLUSIVE SCAN}(\text{Acc}, r_x \cdot r_y \cdot r_z + 1)$ [S1, S2, unofficial].
    *   Initialize top-level cell ranges with scan results: $\text{tlc} \leftarrow \text{INITIALIZE}(\text{Acc}, \text{tlc})$ [S1, S2, unofficial]. This step maps each top-level cell to the start index of its leaf cells in the global leaf array.

4.  **Compute Reference Array Size:**
    *   Initialize an accumulator array $\text{Arc}$ of size $G+1$ (where $G=128$) with zeros [S2, unofficial].
    *   Count leaf references per top-level cell: $\text{Arc} \leftarrow \text{COUNT LEAF REFS}(t, b, r, n, \text{Atl\_p}, \text{tlc})$ [S1, S2, unofficial]. This uses a fast conservative triangle-cell intersection test (bounding box only) [S1, unofficial].
    *   Perform exclusive scan on $\text{Arc}$: $\text{Arc} \leftarrow \text{EXCLUSIVE SCAN}(\text{Arc}, G+1)$ [S1, S2, unofficial].
    *   Get total size $m \leftarrow \text{Arc}[G]$ [S1, S2, unofficial].
    *   Allocate leaf pairs array $\text{Ap}$ of size $m$ [S1, S2, unofficial].

5.  **Fill Reference Array:**
    *   Write pairs $(\text{Leaf Location}, \text{Primitive ID})$ to $\text{Ap}$ [S1, S2, unofficial].
    *   The Leaf Location is computed from the leaf index relative to the top-level cell and the location of the first leaf for that top-level cell. Note: Leaf indices relative to a top-level cell are not unique globally; the global memory location in the leaf array is used as the sort key [S1, unofficial].
    *   Use a more precise intersection test (including plane intersection) when generating these pairs [S1, unofficial].
    *   Invalid intersections are written with a dummy cell index that is discarded during sorting [S1, unofficial].

6.  **Sort:**
    *   Sort $\text{Ap}$ using an improved radix sort (two- and four-way splits) [S1, S2, unofficial]. Sort only on the required bits based on the known maximum key size [S1, unofficial].

7.  **Extract Leaf Cells:**
    *   Extract leaf cell ranges from sorted $\text{Ap}$: $\text{leaves} \leftarrow \text{EXTRACT CELL RANGES}(\text{Ap}, m)$ [S1, S2, unofficial]. This identifies the start and end indices in the Reference Array for each leaf cell.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_two_level_grid(triangles, scene_bounds):
    # 1. Compute Bounds and Resolution
    b = compute_bounds(triangles)
    r = compute_top_level_resolution(b)
    
    # 2. Build Top-Level Grid
    # Build a temporary uniform grid to get top-level overlaps
    data = build_uniform_grid(triangles, b, r)
    atl_p = get_sorted_top_level_pairs(data) # Sorted by cell index
    n = get_number_of_top_level_references(data)
    tlc = get_top_level_cell_ranges(data)
    
    # 3. Compute Leaf Cell Locations
    # Acc size: rx * ry * rz + 1
    acc = array_of_zeros(r.x * r.y * r.z + 1)
    acc = count_leaf_cells(b, r, tlc)
    acc = exclusive_scan(acc)
    tlc = initialize_leaf_cell_locations(acc, tlc)
    
    # 4. Compute Reference Array Size
    # Arc size: 129 (G=128)
    arc = array_of_zeros(129)
    arc = count_leaf_refs(triangles, b, r, n, atl_p, tlc)
    arc = exclusive_scan(arc)
    m = arc[128]
    ap = allocate_array(m)
    
    # 5. Fill Reference Array
    # Write (global_leaf_index, primitive_id) pairs
    # Use conservative test for counting, precise for writing
    write_pairs(ap, triangles, b, r, n, atl_p, tlc, arc)
    
    # 6. Sort
    ap = radix_sort(ap)
    
    # 7. Extract Leaf Cells
    leaves = extract_cell_ranges(ap, m)
    
    return ap, leaves, tlc
```

### Edge Cases and Pitfalls
*   **GPU Memory Constraints:** Dynamic allocation is not possible on GPUs. The size of the Reference Array must be computed conservatively (using bounding box tests) before precise generation to ensure sufficient space [S1, unofficial].
*   **Leaf Index Uniqueness:** Leaf indices relative to a top-level cell are not unique across the entire structure (e.g., multiple top-level cells may have a leaf index 0). Therefore, the global memory location in the leaf array must be used as the sort key for leaf pairs [S1, unofficial].
*   **Resolution Limit:** Using 24 bits for resolution limits the top-level cell resolution to $256^3$ [S1, S2, unofficial].

## Subtopics

### [Two-Level Grid Construction](5226-two-level-grid-construction.md)
A massively parallel, sort-based algorithm for building the structure, solving the problem of constructing high-quality acceleration structures for non-uniform primitive distributions with linear work complexity.

### [Two-Level Grid Ray Traversal](6252-two-level-grid-ray-traversal.md)
A ray-grid traversal algorithm modifying the standard DDA to step through top-level cells and treat each as a uniform grid for leaf traversal, reusing decision variables between levels.

## Operating characteristics
*   **Time Complexity:** Construction is linear with respect to the number of primitives and cells, dominated by the sort operation [S1, unofficial].
*   **Memory Complexity:** The structure uses three arrays. The Reference Array size is determined by the number of primitive-cell overlaps. The Leaf Array size is proportional to the number of leaf cells. The Top-Level Cells Array size is proportional to the top-level grid resolution [S1, S2, unofficial].
*   **Measured Performance:** On an NVIDIA GTX 470, the implementation traced 24 million primary rays/sec for the Conference scene and 22 million for the Fairy scene [S2, unofficial]. This is inferior to state-of-the-art static scene approaches like Aila and Laine [AL09], which reported 142 million and 75 million rays/sec respectively for the same scenes [S2, unofficial].
*   **Strengths:** Handles non-uniform primitive distributions robustly, avoiding the "teapot in a stadium problem" [S1, S2, unofficial]. Efficient construction for dynamic scenes due to parallel sort-based approach [S1, unofficial].
*   **Limitations:** Inferior traversal performance compared to optimized static BVHs or other hierarchical methods [S2, unofficial]. Resolution of top-level cells is limited to $256^3$ due to bit packing [S1, S2, unofficial].

## Background
The two-level grid is a fixed-depth hierarchy introduced by Jevans and Wyvill [JW89] and applied to ray tracing on GPUs by Kalojanov, Billeter, and Slusallek [S1, unofficial]. It builds on the sort-based uniform grid construction by Kalojanov and Slusallek [KS09] and the improved radix sort by Billeter et al. [BOA09]. The method was implemented in CUDA [NBGS08] and published in 2010 [S1, S2, unofficial].

## Sources
[S1] Two-Level Grids for Ray Tracing on GPUs, https://studylib.net/doc/18458638/two-level-grids-for-ray-tracing-on-gpus, fetched 2026-10-25 (unofficial)
[S2] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
