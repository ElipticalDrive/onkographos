# Two-Level Grids for Ray Tracing on GPUs

Also called: Nested Grids, Two-Level Grid, Two-Level Grids for Ray Tracing, Two-Level Nested Grid

## Overview
Two-Level Grids for Ray Tracing on GPUs is a fixed-depth hierarchical acceleration structure designed to handle non-uniform primitive distributions in dynamic scenes, solving the "teapot in a stadium problem" where uniform grids fail to eliminate intersection candidates in sparse regions [S1, unofficial][S2, unofficial]. It consists of a sparse top-level grid where each cell contains a sub-grid (leaf grid) of arbitrary resolution, offering a robust alternative to uniform grids by allowing variable resolution sub-grids [S1, unofficial]. Unlike its sibling [Macrocell Grid Traversal](5207-macrocell-grid-traversal.md), which typically uses a fixed macro-grid with uniform or simple subdivision, the two-level grid explicitly constructs distinct leaf-level uniform grids for each top-level cell, enabling efficient SIMD-friendly traversal that minimizes code divergence [S1, unofficial].

## How it works
The method involves constructing a hierarchical structure via a top-down, sort-based approach on the GPU, followed by a modified DDA traversal algorithm.

### Inputs and Outputs
*   **Input:** A set of triangles $t$, scene bounding box $b$, and top-level resolution $r$.
*   **Output:** A two-level grid structure comprising a Top-Level Cells Array, a Leaf Array, and a Reference Array.

### Data Structures
The structure uses three main arrays [S1, unofficial][S2, unofficial]:

1.  **Top-Level Cells Array:** Stores metadata for each top-level cell. Each entry consists of two 32-bit values:
    *   Bits 0–31: Location (index) of the first leaf cell in the Leaf Array.
    *   Bits 0–23: Cell resolution (limiting resolution to $256^3$).
    *   Bits 24–31: Flags (e.g., 'is empty', 'has children') [S1, unofficial][S2, unofficial].
2.  **Leaf Array:** Stores start and end indices into the Reference Array for each leaf cell. Each leaf cell entry contains two integral values indicating the interval in the Reference Array [S1, unofficial].
3.  **Reference Array:** A single array storing pairs of `(leaf_cell_index, primitive_reference)`. Each primitive is referenced as many times as the number of leaf cells it overlaps. References to primitives intersecting the same leaf cell are stored contiguously [S1, unofficial][S2, unofficial].

### Construction Algorithm
The construction is performed in parallel on the GPU using radix sort [S1, unofficial].

**Parameters:**
*   Grid density $\lambda$: A user-defined constant. Typical value for top-level cells is $\lambda = 1.2$ globally [S1, unofficial]. Top-level density is set to $\lambda = 1/16$ primitives referenced in the leaves to optimize hybrid intersector performance [S1, unofficial].

**Steps:**

1.  **Compute Bounds and Resolution:**
    Compute the bounding box of all primitives $b$ and the top-level grid resolution $r$ [S1, unofficial][S2, unofficial].

2.  **Upload Data:**
    Upload triangle data $t$ to GPU memory [S1, unofficial][S2, unofficial].

3.  **Build Top-Level Uniform Grid:**
    Build a temporary uniform grid using the sort-based approach by Kalojanov and Slusallek [KS09] to initialize top-level cells. Determine all triangle-cell overlaps and store them as pairs of `(Cell Index, Primitive ID)` [S1, unofficial].

4.  **Get Top-Level Pairs and Ranges:**
    *   Get sorted top-level pairs $Atl_p$ from the temporary data [S1, unofficial][S2, unofficial].
    *   Get the number of top-level references $n$ [S1, unofficial][S2, unofficial].
    *   Get top-level cell ranges $tlc$ [S1, unofficial][S2, unofficial].

5.  **Compute Leaf Cell Locations:**
    *   Set grid dimension $G = (r_y, r_z)$ and block size $B = r_x$ [S2, unofficial].
    *   Initialize an accumulator array $Acc$ of size $r_x \cdot r_y \cdot r_z + 1$ with zeros [S1, unofficial][S2, unofficial].
    *   Count leaf cells per top-level cell using bounds, resolution, and top-level cell ranges: $Acc \leftarrow \text{COUNT LEAF CELLS}(b, r, tlc)$ [S1, unofficial][S2, unofficial].
    *   Perform exclusive scan on $Acc$: $Acc \leftarrow \text{EXCLUSIVE SCAN}(Acc, r_x \cdot r_y \cdot r_z + 1)$ [S1, unofficial][S2, unofficial].
    *   Initialize top-level cell ranges with the computed leaf cell locations: $tlc \leftarrow \text{INITIALIZE}(Acc, tlc)$ [S1, unofficial][S2, unofficial].

6.  **Compute Reference Array Size:**
    *   Set grid dimension $G = 128$ and block size $B = 256$ [S2, unofficial].
    *   Initialize an accumulator array $Arc$ of size $G + 1$ with zeros [S1, unofficial][S2, unofficial].
    *   Count leaf references per top-level cell using triangles, bounds, resolution, number of references, sorted pairs, and cell ranges: $Arc \leftarrow \text{COUNT LEAF REFS}(t, b, r, n, Atl_p, tlc)$ [S1, unofficial][S2, unofficial].
    *   Perform exclusive scan on $Arc$: $Arc \leftarrow \text{EXCLUSIVE SCAN}(Arc, G + 1)$ [S1, unofficial][S2, unofficial].
    *   Get total size $m \leftarrow Arc[G]$ [S1, unofficial].
    *   Allocate leaf pairs array $Ap$ of size $m$ [S1, unofficial][S2, unofficial].

7.  **Fill Reference Array:**
    *   Write pairs `(Leaf Location, Prim ID)` to $Ap$. The leaf location is computed from the leaf index and the location of the first leaf for the corresponding top-level cell [S1, unofficial].
    *   Use a fast conservative triangle-cell intersection test (bounding box only) when counting output pairs, and a more precise test (including plane intersection) when generating them [S1, unofficial].
    *   Invalid intersections are written with a dummy cell index discarded during sorting [S1, unofficial].
    *   $Ap \leftarrow \text{WRITE PAIRS}(t, b, r, n, Atl_p, tlc, Arc)$ [S1, unofficial][S2, unofficial].

8.  **Sort Reference Array:**
    *   Sort $Ap$ using an improved radix sort by Billeter et al. [BOA09] with two- and four-way splits. Sort only on the required bits (up to two bits at a time) based on the known maximum key size [S1, unofficial][S2, unofficial].
    *   $Ap \leftarrow \text{SORT}(Ap)$ [S1, unofficial][S2, unofficial].

9.  **Extract Leaf Cell Ranges:**
    *   Extract cell ranges from the sorted reference array to finalize leaf cell intervals: $leaves \leftarrow \text{EXTRACT CELL RANGES}(Ap, m)$ [S1, unofficial][S2, unofficial].
    *   Check in parallel for range borders (neighboring pairs with different keys) to determine start and end indices for each leaf cell [S1, unofficial].
    *   Extract the reference array out of the pairs array to compact the data and free space after outputting the leaf array [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_two_level_grid(triangles, scene_bounds, top_level_res):
    # 1. Compute bounds and resolution
    b = compute_bounds(triangles)
    r = compute_top_level_resolution(scene_bounds)
    
    # 2. Upload triangles
    t = upload_to_gpu(triangles)
    
    # 3. Build top-level uniform grid
    data = build_uniform_grid(t, b, r)
    
    # 4. Get top-level data
    atl_p = get_sorted_top_level_pairs(data)
    n = get_number_of_top_level_references(data)
    tlc = get_top_level_cell_ranges(data)
    
    # 5. Compute leaf cell locations
    G = (r.y, r.z)
    B = r.x
    acc = array_of_zeros(r.x * r.y * r.z + 1)
    acc = count_leaf_cells(b, r, tlc, G, B)
    acc = exclusive_scan(acc, r.x * r.y * r.z + 1)
    tlc = initialize_top_level_cell_ranges(acc, tlc, G, B)
    
    # 6. Compute reference array size
    G_ref = 128
    B_ref = 256
    arc = array_of_zeros(G_ref + 1)
    arc = count_leaf_refs(t, b, r, n, atl_p, tlc, G_ref, B_ref)
    arc = exclusive_scan(arc, G_ref + 1)
    m = arc[G_ref]
    ap = allocate_array(m)
    
    # 7. Fill reference array
    # Use conservative test for counting, precise for writing
    ap = write_pairs(t, b, r, n, atl_p, tlc, arc, G_ref, B_ref)
    
    # 8. Sort reference array
    ap = radix_sort(ap)
    
    # 9. Extract leaf cells and reference array
    leaves = extract_cell_ranges(ap, m)
    ref_array = compact_reference_array(ap, leaves)
    
    return TopLevelGrid(tlc), LeafArray(leaves), ReferenceArray(ref_array)
```

### Ray Traversal
The traversal modifies the classic DDA algorithm by Amanatides and Woo [AW87] [S1, unofficial].
1.  Step through the top-level structure as with a uniform grid.
2.  Treat each top cell as a uniform grid itself and use the same DDA algorithm for leaf traversal [S1, unofficial].
3.  This approach leverages coherence within top-level cells, offering a SIMD-friendly approach that minimizes code divergence except when rays disagree on the traversal level [S1, unofficial].

### Edge Cases and Pitfalls
*   **GPU Memory Constraints:** Pre-allocation of the reference array is required. The size must be computed conservatively (using bounding box tests) before precise generation to ensure sufficient space [S1, unofficial].
*   **Leaf Index Uniqueness:** Leaf indices relative to a top-level cell are not unique across the entire structure (e.g., multiple top-level cells may have a leaf index 0). Therefore, the global memory location in the leaf array must be used as the sort key for leaf pairs [S1, unofficial].
*   **Resolution Limits:** Using 24 bits for resolution limits the resolution of each top-level cell to $256^3$, though this is implementation-specific [S1, unofficial].
*   **Density Selection:** Choosing different density or resolution for different top-level cells provides higher quality but makes construction too slow [S1, unofficial].

## Subtopics

### [Two-Level Grid Ray Traversal](6252-two-level-grid-ray-traversal.md)
Two-Level Grid Ray Traversal is a hierarchical ray traversal algorithm designed for two-level grid structures, modifying the classic DDA algorithm to step through top-level cells and then treat each top cell as a uniform grid for leaf traversal. It solves the problem of efficiently traversing sparse or complex volumetric data by leveraging coherence within top-level cells, offering a SIMD-friendly approach that minimizes code divergence except when rays disagree on the traversal level.

### Two-Level Grid Construction
Two-Level Grid Construction is a construction algorithm for two-level acceleration structures that uses radix sort to build top-level and leaf-level grids efficiently on GPUs, solving the problem of fast, linear-complexity build times for large scenes [S1, unofficial]. It sorts pairs of `(key, primitive_id)` using radix sort, considering only required bits (up to two bits at a time) based on the known size of the largest key [S1, unofficial]. It computes leaf cells by checking in parallel for range borders, i.e., neighboring pairs with different keys in the sorted array [S1, unofficial]. It extracts the reference array out of the pairs array to compact the data and free space after outputting the leaf array [S1, unofficial]. The work complexity is linear, provided cell resolution can be determined in constant time knowing the number of overlapped primitives and the extent of the bounding box of each top-level cell [S1, unofficial]. A typical parameter is grid density $\lambda = 1.2$ globally across all cells, with top-level density set to $\lambda = 1/16$ primitives referenced in the leaves to optimize hybrid intersector performance [S1, unofficial]. Choosing different density or resolution for different top-level cells provides higher quality but makes construction too slow [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The construction work complexity is linear, provided cell resolution can be determined in constant time [S1, unofficial].
*   **Memory Complexity:** Requires pre-allocation of the reference array, which can be large. The structure uses three arrays: Reference, Leaf, and Top-Level Cells [S1, unofficial].
*   **Measured Performance:** On a GTX 470, the implementation traced 24 million primary rays/sec for the Conference scene and 22 million for the Fairy scene [S2, unofficial]. This is inferior to state-of-the-art approaches for static scenes (specifically Aila and Laine [AL09]), which reported 142 million and 75 million respectively [S2, unofficial].
*   **Strengths:** Handles non-uniform primitive distributions more robustly than uniform grids; fast, linear-complexity build times; SIMD-friendly traversal [S1, unofficial].
*   **Limitations:** Inferior traversal performance compared to static scene optimizations; GPU memory constraints require careful pre-allocation; resolution limited to $256^3$ per top-level cell due to bit packing [S1, unofficial][S2, unofficial].
*   **When to Use:** Dynamic scenes with non-uniform primitive distributions where fast construction is prioritized over maximum traversal speed [S1, unofficial].
*   **When Not to Use:** Static scenes where higher traversal performance is critical and more complex build algorithms are acceptable [S2, unofficial].

## Background
The two-level grid is a fixed-depth hierarchy introduced by Jevans and Wyvill [JW89] as a member of a class of hierarchical grids [S1, unofficial]. Its application to ray tracing on GPUs was first studied by Kalojanov, Billeter, and Slusallek in 2010 [S1, unofficial]. The construction algorithm builds upon the sort-based uniform grid construction by Kalojanov and Slusallek [KS09] and uses an improved radix sort by Billeter et al. [BOA09] [S1, unofficial].

## Sources
[S1] Two-Level Grids for Ray Tracing on GPUs, https://studylib.net/doc/18458638/two-level-grids-for-ray-tracing-on-gpus, fetched 2026-10-25 (unofficial)
[S2] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
