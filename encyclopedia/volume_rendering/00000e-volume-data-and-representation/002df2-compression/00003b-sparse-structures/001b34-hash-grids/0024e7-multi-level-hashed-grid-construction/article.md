# Multi-Level Hashed Grid Construction

Also called: Grid Encoding (Hash Grid), Hash Grid, Hashed Grid with Macrocells, Multi-Level Hashed Grid, Multi-level Hash Tables, Multi-level hashed grid implementation, Row Displacement Compressed Grid

## Overview
Multi-Level Hashed Grid Construction is a method for rapidly building an acceleration structure that combines the render-time performance of multi-level grids with the low memory footprint of hashed grids [S1, S2, unofficial]. It solves the problem of high memory consumption in classic multi-level array grids and the poor traversal performance of one-level hashed grids for large, sparse scanned scenes [S1, S2, unofficial]. It differs from [Hash Grids](936-hash-grids.md) by adding macrocells to skip empty regions during traversal, and from [Voxel World Data Structure](1101-voxel-world-data-structure.md) by using row displacement compression to eliminate empty cells entirely [S1, S2, unofficial].

## How it works
The method constructs a sparse acceleration structure in three phases: heuristic grid sizing, hashed grid construction via row displacement compression, and macrocell generation [S1, S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Scene primitives (e.g., triangles), grid density parameter $\rho$, macrocell downscaling factor $D_{scale}$.
*   **Outputs:**
    *   Hash Table $H$: Array of integers storing item list start indices.
    *   Item List $L$: Linear array of primitive indices.
    *   Offset Array $O$: Array of row offsets for the hash function.
    *   Macrocells: 3D bit arrays indicating empty regions at multiple resolutions.

### Data Structures
1.  **Offset Array $O$**: Stores the starting index in $H$ for each row $(y, z)$. Size $M_y \times M_z$.
2.  **Hash Table $H$**: Array of size $N_H$. Initially stores counts of items per cell; after prefix sum, stores the starting index of the item list for each cell.
3.  **Item List $L$**: Linear array of size $\sum H_{counts}$. Stores primitive indices.
4.  **Temporary Bit Array $b_H$**: Used during row displacement to find non-colliding offsets. Size $2 \times N_{non-empty}$.
5.  **Macrocell Bitmap $D$**: 3D bit array indicating if a region is empty.

### Parameters
*   $\rho$ (Grid Density): Empirically chosen as 32 for the Buddha scene to balance render time and build time [S1, S2, unofficial].
*   $D_{scale}$ (Macrocell Downscaling): Factor of 6 per extent, determined empirically [S1, S2, unofficial].

### Algorithm Steps

1.  **Grid Dimensioning**: Determine grid dimensions $(M_x, M_y, M_z)$ using Heuristic 1 with density $\rho$ [S1, S2, unofficial].
2.  **Row Offset Computation**: Compute offsets for each row $(y, z)$ to map 2D rows to 1D indices in $O$ [S1, S2, unofficial]:
    $$o(y, z) = (M_y \times z) + y$$
    Store $o(y, z)$ in $O[o(y, z)]$ [S1, S2, unofficial].
3.  **Row Displacement Compression**:
    *   Initialize temporary bit array $b_H$ of size $2 \times N_{non-empty}$ to zero [S2, unofficial].
    *   For each row $(y, z)$, find the smallest offset starting from the previous row's offset.
    *   Check for bit collisions between the current offset and $b_H$. If collision, increment offset and test again [S2, unofficial].
    *   Mark bits in $b_H$ for the chosen offset [S2, unofficial].
    *   Allocate Hash Table $H$ with size $N_H$ equal to the position of the last non-empty bit in $b_H$ plus one. Initialize $H$ to zero [S1, S2, unofficial].
4.  **Item Counting**:
    *   For each item $i$ whose bounding box intersects cell $j = (x, y, z)$, compute hash index:
        $$h(x, y, z) = O[(M_y \times z) + y] + x$$
    *   Increment $H[h(x, y, z)]$ [S1, S2, unofficial].
5.  **Prefix Sum**:
    *   Compute prefix sums on $H$ to determine list start indices:
        ```python
        for i in range(1, N_H + 1):
            H[i] += H[i - 1]
        ```
    *   The total number of items is $H[N_H - 1]$ [S1, S2, unofficial].
6.  **Item List Construction**:
    *   Allocate Item List $L$ of size $H[N_H - 1]$ [S1, S2, unofficial].
    *   Insert items in reverse order to maintain sorted indices:
        ```python
        for i in range(N_I - 1, -1, -1):
            for each cell j intersected by object i:
                H[j] -= 1
                L[H[j]] = i
        ```
    *   After this, $H[j]$ points to the first item of cell $j$'s list in $L$ [S1, S2, unofficial].
7.  **Macrocell Construction**:
    *   Overlay a coarser grid to skip empty cells [S1, S2, unofficial].
    *   Downscale the 3D bitmap $D$ (representing non-empty cells) by factor $D_{scale}=6$ on each extent using a 3D bitmap scaling operation [S1, S2, unofficial].
    *   Store the resulting bit arrays as macrocells for traversal [S1, S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_multi_level_hashed_grid(primitives, M_x, M_y, M_z, rho, D_scale):
    # Step 1: Compute Row Offsets
    O = [0] * (M_y * M_z)
    for z in range(M_z):
        for y in range(M_y):
            idx = (M_y * z) + y
            O[idx] = 0 # Placeholder, refined in displacement

    # Step 2: Row Displacement Compression
    # Determine non-empty cells first to size b_H
    non_empty_cells = set()
    for prim in primitives:
        for cell in intersect_cells(prim, M_x, M_y, M_z):
            non_empty_cells.add(cell)
    
    N_non_empty = len(non_empty_cells)
    b_H = [0] * (2 * N_non_empty) # Temporary bit array
    H_counts = [0] * (M_x * M_y * M_z) # Temporary counts
    
    # Map rows to offsets
    row_offsets = {}
    current_offset = 0
    for z in range(M_z):
        for y in range(M_y):
            row_key = (y, z)
            # Find smallest offset not in b_H
            while b_H[current_offset]:
                current_offset += 1
            row_offsets[row_key] = current_offset
            # Mark bits used by this row's potential cells? 
            # Note: The source describes marking bits in b_H for the offset itself 
            # to prevent other rows from using it.
            # We assume b_H tracks occupied offset slots.
            b_H[current_offset] = 1 
            
    # Allocate H based on max offset
    N_H = current_offset + 1
    H = [0] * N_H
    
    # Step 3: Count Items
    for prim in primitives:
        for cell in intersect_cells(prim, M_x, M_y, M_z):
            x, y, z = cell
            o_yz = row_offsets[(y, z)]
            h_idx = o_yz + x
            H[h_idx] += 1
            
    # Step 4: Prefix Sum
    for i in range(1, N_H):
        H[i] += H[i - 1]
        
    # Step 5: Insert Items
    L = [0] * H[N_H - 1]
    for i in range(len(primitives) - 1, -1, -1):
        for cell in intersect_cells(primitives[i], M_x, M_y, M_z):
            x, y, z = cell
            o_yz = row_offsets[(y, z)]
            h_idx = o_yz + x
            H[h_idx] -= 1
            L[H[h_idx]] = i
            
    # Step 6: Macrocells
    # Downscale bitmap D by D_scale
    macrocells = downscale_bitmap(D, D_scale)
    
    return H, L, O, macrocells
```

### Edge Cases and Pitfalls
*   **Memory Exhaustion**: Non-compressed multi-level array grids may exhaust system memory for large scenes (e.g., Lucy scene), whereas hashed grids reduce memory by an order of magnitude [S1, S2, unofficial].
*   **Hash Collision**: Row displacement ensures no collisions for static data, but requires careful bit array management [S1, S2, unofficial].
*   **Performance Degradation**: For small non-uniform density scenes (e.g., Office, Conference), classic adaptive multi-level array grids may perform better than multi-level hashed grids [S1, S2, unofficial].

## Subtopics

### [Multi-Grid Merging](7905-multi-grid-merging.md)
Multi-Grid Merging combines two sparse hash grids into a single structure by taking the union of their active voxels. It differs from [Replace Compositing](7905-multi-grid-merging.md) by resolving overlaps via a specific aggregation function (typically minimum, maximum, or sum) rather than overwriting, preserving information from both inputs. This method is essential for building multi-level representations or combining data from different sources without losing overlapping details.

### [Neuralangelo Hashgrid Configuration](8523-neuralangelo-hashgrid-configuration.md)
Neuralangelo Hashgrid Configuration defines the specific hyperparameters for the multi-level hashed grid construction used in the Neuralangelo neural implicit surface reconstruction method. It solves the problem of balancing representational capacity and memory usage for high-fidelity 3D reconstruction by specifying a fixed set of grid resolutions, feature dimensions, and hash map sizes. It is distinguished from generic hash grid implementations by its specific adherence to the Sdfstudio baseline configuration, which prioritizes fine-grained detail recovery for indoor scenes.

### [Hierarchical Voxel Block Hashing](12402-hierarchical-voxel-block-hashing.md)
Hierarchical Voxel Block Hashing (HVBH) is a sparse volumetric representation that combines the memory efficiency of hash-based block indexing with the adaptability of multi-resolution grids. Unlike standard voxel block hashing which uses a single resolution, HVBH maintains a fixed hierarchy of $L$ levels, allowing coarse blocks to point to finer sub-blocks only where detail is required. This approach bypasses the time-consuming tree traversal of octrees while avoiding the memory waste of uniform high-resolution grids, achieving state-of-the-art accuracy with approximately 40% memory savings compared to fixed grids.

### Multigrid Volume Compression Header Generation
Algorithm to generate compression headers for a multigrid volume representation. It determines the dominant label and constant subregion status for each node in the hierarchy by analyzing child elements [S5, unofficial].
1.  Initialize constant flag to true. Iterate through 8 child elements. Skip children with label 0xFFFFFFFF (outside volume) [S5, unofficial].
2.  Count occurrences of each label among the 8 children. Track the max_label and max_ocurrences. Update max_label if current child's label has higher occurrence count [S5, unofficial].
3.  Update constant flag: It remains true only if all children have constant_subregion set (or are single elements) AND all children share the same label as the first child [S5, unofficial].
4.  Write the computed max_label and constant flag to the output array at the position corresponding to the current LOD and position [S5, unofficial].
5.  Update lod_start for the next level by adding current_dim^3. Assert that lod_start equals out.size() at the end to ensure correct memory allocation [S5, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear in the number of scene items $N$, except for hash function computation which has a worst-case time complexity of $O(M \cdot 4/3)$, where $M$ is the number of cells in the grid [S1, S2, unofficial].
*   **Memory Complexity**: Roughly one order of magnitude lower than regular non-compressed grids [S1, S2, unofficial]. Slightly higher than one-level hashed grids due to macrocells [S1, S2, unofficial].
*   **Performance**: On the Thai Statue scene (1024x1024), multi-level hashed grid render time is 0.94s vs 1.60s for one-level hashed grid, a 52% improvement over Lagae and Dutre (2008) baseline [S2, unofficial]. Multi-level hashed grids provide around twice the render time performance of one-level hashed grids for large scanned scenes with many empty cells [S1, S2, unofficial].
*   **Strengths**: Low build time, low memory footprint, fast traversal for large sparse scenes [S1, S2, unofficial].
*   **Limitations**: Poorer performance than adaptive multi-level array grids for small non-uniform density scenes [S1, S2, unofficial]. Storage cost increases as video length increases in dynamic applications [S4].

## Background
The method was introduced to address the limitations of classic multi-level array grids (high memory) and one-level hashed grids (slow traversal) [S1, S2, unofficial]. It builds on row displacement compression (Lagae and Dutre, 2008) and multi-level macrocells (Wald et al., 2006) [S1, S2, unofficial]. It is used in interactive ray tracing of large scanned models [S1, unofficial] and in neural implicit surface reconstruction (Neuralangelo) [S3, unofficial] and dynamic volumetric video representation [S4].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28153/, fetched 2026-10-24 (unofficial)
[S2] PDF Multi-level Grid Strategies for Ray Tracing, https://pdfs.semanticscholar.org/f922/40132d0d7340ff97ac4074b0a6906fb44c44.pdf, fetched 2026-10-25 (unofficial)
[S3] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
[S4] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S5] max-pio/compressed-segmentation-volumes, https://github.com/max-pio/compressed-segmentation-volumes, fetched 2026-10-26 (unofficial)
