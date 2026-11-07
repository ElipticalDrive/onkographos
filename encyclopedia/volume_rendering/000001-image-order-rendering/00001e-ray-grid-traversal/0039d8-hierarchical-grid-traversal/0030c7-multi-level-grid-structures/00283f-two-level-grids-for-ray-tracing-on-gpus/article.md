# Two-Level Grids for Ray Tracing on GPUs

Also called: Nested Grids, Two-Level Grid, Two-Level Grids for Ray Tracing, Two-Level Nested Grid

## Overview
Two-Level Grids for Ray Tracing on GPUs is a fixed-depth hierarchical acceleration structure designed to handle non-uniform primitive distributions in dynamic scenes more robustly than uniform grids [S1, S2, unofficial]. It solves the "teapot in a stadium problem" by using a sparse top-level grid where each cell contains a sub-grid (leaf grid) of arbitrary resolution [S1, S2, unofficial]. Unlike its siblings, it avoids pointer chasing by using contiguous arrays and radix sort for construction, making it highly suitable for GPU parallelism [S1, S2, unofficial].

## How it works
**Inputs:** Triangle mesh data (positions, indices), scene bounding box $b$, top-level resolution $r$, and grid density parameter $\lambda$.
**Outputs:** Reference Array, Leaf Array, and Top-Level Cells Array.

### Data Structures
The structure consists of three main arrays [S1, S2, unofficial]:

1.  **Top-Level Cells Array:** An array of size $N_{top} = r_x \times r_y \times r_z$. Each entry is a 64-bit value (two 32-bit integers) [S1, S2, unofficial]:
    *   **First 32 bits:** The index of the first leaf cell in the Leaf Array.
    *   **Second 32 bits:**
        *   **Lower 24 bits:** The resolution of the leaf grid in this cell (limiting resolution to $256^3$) [S1, S2, unofficial].
        *   **Upper 8 bits:** Flags (e.g., 'is empty', 'has children') [S1, S2, unofficial].

2.  **Leaf Array:** An array of size $N_{leaves}$. Each entry stores two 32-bit integers:
    *   **Start Index:** The starting index in the Reference Array for primitives overlapping this leaf cell.
    *   **End Index:** The ending index (exclusive) in the Reference Array [S1, S2, unofficial].

3.  **Reference Array:** A contiguous array storing pairs of `(Leaf Cell Index, Primitive ID)`. Each primitive is referenced once for every leaf cell it overlaps. References for the same leaf cell are stored contiguously [S1, S2, unofficial].

### Construction Algorithm
The construction is data-parallel and uses radix sort [S1, S2, unofficial].

1.  **Compute Bounds and Resolution:**
    *   Compute the scene bounding box $b$ [S1, S2, unofficial].
    *   Compute the top-level resolution $r$ based on the grid density $\lambda$ [S1, S2, unofficial]. Typical value: $\lambda = 1.2$ primitives per top-level cell [S1, unofficial]. For hybrid intersector optimization, top-level density may be set to $\lambda = 1/16$ [S1, unofficial].

2.  **Build Top-Level Grid:**
    *   Upload triangles $t$ to GPU [S1, S2, unofficial].
    *   Build a temporary uniform grid using the sort-based approach by Kalojanov and Slusallek [KS09] [S1, S2, unofficial].
    *   Determine all triangle-cell overlaps and store as pairs `(Cell Index, Primitive ID)` [S1, S2, unofficial].
    *   Sort these pairs using cell indices as keys [S1, S2, unofficial].
    *   Extract sorted top-level pairs $Atl_p$, number of references $n$, and top-level cell ranges $tlc$ [S1, S2, unofficial].

3.  **Compute Leaf Cell Locations:**
    *   Initialize an accumulator array $Acc$ of size $r_x \times r_y \times r_z + 1$ with zeros [S1, S2, unofficial].
    *   Count leaf cells per top-level cell using bounds $b$, resolution $r$, and ranges $tlc$: $Acc \leftarrow \text{COUNT LEAF CELLS}(b, r, tlc)$ [S1, S2, unofficial].
    *   Perform an exclusive scan on $Acc$: $Acc \leftarrow \text{EXCLUSIVE SCAN}(Acc, r_x r_y r_z + 1)$ [S1, S2, unofficial].
    *   Initialize top-level cell ranges with the scan results to determine the start index of leaf cells in the Leaf Array: $tlc \leftarrow \text{INITIALIZE}(Acc, tlc)$ [S1, S2, unofficial].

4.  **Compute Reference Array Size:**
    *   Initialize an accumulator array $Arc$ of size $G+1$ (where $G=128$) with zeros [S1, S2, unofficial].
    *   Count leaf references per top-level cell using triangles $t$, bounds $b$, resolution $r$, number of references $n$, sorted pairs $Atl_p$, and cell ranges $tlc$: $Arc \leftarrow \text{COUNT LEAF REFS}(t, b, r, n, Atl_p, tlc)$ [S1, S2, unofficial].
    *   Perform an exclusive scan on $Arc$: $Arc \leftarrow \text{EXCLUSIVE SCAN}(Arc, G+1)$ [S1, S2, unofficial].
    *   Get total size $m \leftarrow Arc[G]$ [S1, S2, unofficial].
    *   Allocate leaf pairs array $Ap$ of size $m$ [S1, S2, unofficial].

5.  **Fill Reference Array:**
    *   For each primitive, compute its global leaf cell index. Note: Leaf indices relative to a top-level cell are not unique. The global memory location in the leaf array is used as the sort key [S1, S2, unofficial].
    *   Use a fast conservative triangle-cell intersection test (bounding box only) when counting output pairs, and a more precise test (plane intersection) when generating them [S1, S2, unofficial].
    *   Write pairs `(Leaf Location, Primitive ID)` to $Ap$ [S1, S2, unofficial]. Invalid intersections are written with a dummy cell index discarded during sorting [S1, S2, unofficial].

6.  **Sort and Extract:**
    *   Sort $Ap$ using an improved radix sort by Billeter et al. [BOA09] with two- and four-way splits, sorting only on required bits [S1, S2, unofficial].
    *   Extract leaf cell ranges from the sorted $Ap$ to finalize leaf cell intervals: $leaves \leftarrow \text{EXTRACT CELL RANGES}(Ap, m)$ [S1, S2, unofficial].
    *   Extract the reference array out of the pairs array to compact the data [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_two_level_grid(triangles, lambda_density=1.2):
    # 1. Compute Bounds and Resolution
    b = compute_bounds(triangles)
    r = compute_top_level_resolution(b, lambda_density)
    
    # 2. Build Top-Level Grid
    # Build temporary uniform grid and get sorted pairs (Cell Index, Prim ID)
    data = build_uniform_grid(triangles, b, r)
    atl_p = get_sorted_top_level_pairs(data)
    n = get_number_of_top_level_references(data)
    tlc = get_top_level_cell_ranges(data)
    
    # 3. Compute Leaf Cell Locations
    # Acc size: rx * ry * rz + 1
    acc = array_of_zeros(r[0] * r[1] * r[2] + 1)
    acc = count_leaf_cells(b, r, tlc, acc)
    acc = exclusive_scan(acc)
    tlc = initialize_top_level_cell_ranges(acc, tlc)
    
    # 4. Compute Reference Array Size
    # Arc size: G + 1, G=128
    arc = array_of_zeros(129)
    arc = count_leaf_refs(triangles, b, r, n, atl_p, tlc, arc)
    arc = exclusive_scan(arc)
    m = arc[128]
    ap = allocate_array(m)
    
    # 5. Fill Reference Array
    # Compute global leaf location for each primitive
    # Use conservative test for counting, precise for generation
    ap = write_pairs(triangles, b, r, n, atl_p, tlc, arc, ap)
    
    # 6. Sort and Extract
    ap = radix_sort(ap)
    leaves = extract_cell_ranges(ap, m)
    
    return ap, leaves, tlc
```

### Edge Cases and Pitfalls
*   **GPU Memory Constraints:** Dynamic allocation is not possible on GPUs. The size of the reference array must be computed conservatively (using bounding box tests) before precise generation to ensure sufficient space [S1, S2, unofficial].
*   **Leaf Index Uniqueness:** Leaf indices relative to a top-level cell are not unique across the entire structure (e.g., multiple top-level cells may have a leaf index 0). Therefore, the global memory location in the leaf array must be used as the sort key for leaf pairs [S1, S2, unofficial].
*   **Resolution Limit:** Using 24 bits for resolution limits the leaf grid resolution to $256^3$ [S1, S2, unofficial].
*   **Density Choice:** Choosing different density or resolution for different top-level cells provides higher quality but makes construction too slow [S1, unofficial].

## Subtopics

### [Two-Level Grid Ray Traversal](6252-two-level-grid-ray-traversal.md)
Two-Level Grid Ray Traversal is a hierarchical ray traversal algorithm designed for two-level grid structures, modifying the classic DDA algorithm to step through top-level cells and then treat each top cell as a uniform grid for leaf traversal. It solves the problem of efficiently traversing sparse or complex volumetric data by leveraging coherence within top-level cells, offering a SIMD-friendly approach that minimizes code divergence except when rays disagree on the traversal level.

### Two-Level Grid Construction
Two-Level Grid Construction is a construction algorithm for two-level acceleration structures that uses radix sort to build top-level and leaf-level grids efficiently on GPUs, solving the problem of fast, linear-complexity build times for large scenes [S1, unofficial]. It sorts pairs of (key, primitive_id) using radix sort, considering only required bits (up to two bits at a time) based on the known size of the largest key [S1, unofficial]. It computes leaf cells by checking in parallel for range borders, i.e., neighboring pairs with different keys in the sorted array [S1, unofficial]. It extracts the reference array out of the pairs array to compact the data and free space after outputting the leaf array [S1, unofficial]. Grid density $\lambda$ is a user-defined constant used to determine grid resolution, typically $\lambda = 1.2$ globally [S1, unofficial]. Top-level density may be set to $\lambda = 1/16$ primitives referenced in the leaves to optimize hybrid intersector performance [S1, unofficial]. Work complexity is linear, provided cell resolution can be determined in constant time knowing the number of overlapped primitives and the extent of the bounding box of each top-level cell [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Construction work complexity is linear [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the number of primitives and grid cells [S1, unofficial].
*   **Performance:** On a GTX 470, the implementation traced 24 million primary rays/sec for the Conference scene and 22 million for the Fairy scene [S2, unofficial]. This is inferior to state-of-the-art approaches for static scenes (specifically Aila and Laine [AL09]), which reported 142 million and 75 million respectively [S2, unofficial].
*   **Strengths:** Handles non-uniform primitive distributions robustly; fast, linear-complexity construction on GPUs; SIMD-friendly traversal [S1, S2, unofficial].
*   **Limitations:** Inferior traversal performance compared to optimized static scene structures; resolution limited to $256^3$ per leaf cell due to bit packing [S1, S2, unofficial].
*   **When to use:** Dynamic scenes with non-uniform primitive distributions where fast construction is critical [S1, unofficial].
*   **When not to use:** Static scenes where maximum traversal performance is required [S2, unofficial].

## Background
The two-level grid is a fixed-depth hierarchy introduced by Jevans and Wyvill [JW89] [S1, unofficial]. Its application to ray tracing on GPUs was studied by Kalojanov, Billeter, and Slusallek [S1, S2, unofficial]. The construction algorithm generalizes the sort-based uniform grid construction by Kalojanov and Slusallek [KS09] and uses radix sort by Billeter et al. [BOA09] [S1, S2, unofficial].

## Sources
[S1] Two-Level Grids for Ray Tracing on GPUs, https://studylib.net/doc/18458638/two-level-grids-for-ray-tracing-on-gpus, fetched 2026-10-25 (unofficial)
[S2] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
