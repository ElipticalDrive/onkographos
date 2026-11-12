# Hash Grids

Also called: Coordinate Hashing, Vertex Hashing for Voxel Grid, Perfect Hashing via Row Displacement, Sparse Grid with Row Displacement

## Overview
Hash Grids are a sparse data structure for volume rendering that maps 3D spatial coordinates to a compact, collision-free hash table using a perfect hash function derived from row displacement compression [S1][S2, S3, unofficial]. Unlike standard open-addressing hash tables that store keys or handle collisions via chaining, this method computes a static offset for each spatial dimension (row) such that non-zero elements map to unique indices in a linear array [S1]. This eliminates the need for collision resolution logic during rendering, allowing for $O(1)$ access to voxel data while storing only occupied cells [S2, S3, unofficial]. It differs from [SparseGrid](904-sparsegrid.md) and [Tree-based structures](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md) by avoiding pointer indirection and tree traversal overhead, offering lower memory overhead for static scenes at the cost of static construction time [S1][S2, unofficial].

## How it works
**Inputs:**
- A set of volumetric items (e.g., triangles, particles, or voxel centers) with 3D positions.
- Scene bounding box (min/max coordinates).
- Grid dimensions $M_x, M_y, M_z$ (determined by a heuristic based on item count and bounding box volume) [S2, S3, unofficial].

**Outputs:**
- Four static arrays: $L$ (Item List), $H$ (Hash Table), $O$ (Offset Table), and $D$ (Domain Bits) [S2, S3, unofficial].

**Data Structures:**
1.  **Array $L$ (Item List):** 1D array of machine words (e.g., `uint32` or `uint64`) containing the actual data (indices, values, or pointers) for all non-empty cells [S2, S3, unofficial].
2.  **Array $H$ (Hash Table):** 1D array of machine words. $H[i]$ stores the starting index in $L$ for the cell corresponding to grid coordinate $i$. The size of the item list for cell $i$ is given by $H[i+1] - H[i]$ (assuming $H$ is sorted or padded) [S2, S3, unofficial]. *Note: In the perfect hash context, $H$ effectively maps the linearized grid index to the offset in $L$.*
3.  **Array $O$ (Offset Table):** 1D array of size $M_y \times M_z$. $O[y \cdot M_z + z]$ stores the displacement offset for the row at $(y, z)$ [S1][S2, S3, unofficial].
4.  **Array $D$ (Domain Bits):** 1D bit array of size $N = M_x \times M_y \times M_z$. $D[k] = 1$ if the cell at linear index $k$ contains data, $0$ otherwise [S2, S3, unofficial].

**Coordinate Mapping:**
3D coordinates $(x, y, z)$ are mapped to a 1D linear index $d(x, y, z)$ using row-major ordering [S2, S3, unofficial]:
$$ d(x, y, z) = ((M_y \times z) + y) \times M_x + x $$

**Construction Algorithm (Row Displacement Compression):**
The goal is to compute offsets $O$ such that for every non-zero element $C(i, j)$ in a conceptual sparse matrix $C$ (where $i$ is the row index $(y,z)$ and $j$ is the column index $x$), the hash function $h(i, j) = O[i] + j$ maps to a unique position in $H$ without collisions [S1].

1.  **Compute Bounding Box:** Determine the scene's bounding box and calculate grid dimensions $M_x, M_y, M_z$ [S2, S3, unofficial].
2.  **Initialize Domain Bits:** Allocate $D$ of size $N = M_x M_y M_z$. Initialize all bits to 0.
3.  **Mark Occupied Cells:** For each item, determine which grid cells its bounding box intersects. For each intersected cell with linear index $k$, set $D[k] = 1$ [S2, S3, unofficial].
4.  **Allocate Offset Table:** Allocate $O$ of size $M_y \times M_z$ [S2, S3, unofficial].
5.  **Allocate Temporary Buffer:** Allocate a temporary bit array $bH$ of size $2 \times (\text{number of non-empty cells})$, initialized to 0. This buffer helps track occupied slots in the hash table during displacement [S2, S3, unofficial].
6.  **Compute Offsets (Row Displacement):**
    - Iterate through each row $i$ (from $0$ to $M_y M_z - 1$).
    - For row $i$, identify the set of column indices $J_i = \{ x \mid D[d(x, i_y, i_z)] = 1 \}$, where $i = i_y M_z + i_z$.
    - Find the smallest offset $O[i]$ such that for all $x \in J_i$, the target hash index $H_{idx} = O[i] + x$ is not marked in $bH$.
    - Mark all $bH[O[i] + x]$ as 1 for $x \in J_i$.
    - Record $O[i]$ in the offset table [S1][S2, S3, unofficial].
7.  **Populate Hash Table $H$:**
    - Initialize $H$ with zeros or appropriate defaults.
    - For each row $i$ and each occupied column $x \in J_i$:
        - Calculate hash index $h = O[i] + x$.
        - Store the item data index (from $L$) at $H[h]$ [S1][S2, S3, unofficial].
8.  **Populate Item List $L$:**
    - $L$ contains the actual data values. The mapping from grid cell to data is $L[H[d(x,y,z)]]$ if $D[d(x,y,z)] == 1$ [S2, S3, unofficial].

**Lookup:**
To retrieve data for coordinate $(x, y, z)$:
1.  Compute linear index $k = d(x, y, z)$.
2.  If $D[k] == 0$, the cell is empty.
3.  If $D[k] == 1$, retrieve the offset $O[y \cdot M_z + z]$.
4.  Compute hash index $h = O[y \cdot M_z + z] + x$.
5.  Retrieve data from $L[H[h]]$ (or directly from $H$ if $H$ stores values, depending on implementation variant) [S1][S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_hash_grid(items, grid_dims):
    Mx, My, Mz = grid_dims
    N = Mx * My * Mz
    
    # 1. Initialize Domain Bits D
    D = [0] * N
    
    # 2. Mark occupied cells
    # Assuming items have a method to return intersecting grid cells
    occupied_cells = set()
    for item in items:
        cells = get_intersecting_cells(item, grid_dims)
        for (x, y, z) in cells:
            k = ((My * z) + y) * Mx + x
            D[k] = 1
            occupied_cells.add(k)
            
    num_occupied = sum(D)
    
    # 3. Allocate Offset Table O
    O = [0] * (My * Mz)
    
    # 4. Allocate temporary bit array bH
    # Size is double the number of non-empty cells to allow displacement
    bH = [0] * (2 * num_occupied)
    
    # 5. Compute Offsets using Row Displacement
    for row_idx in range(My * Mz):
        y = row_idx // Mz
        z = row_idx % Mz
        
        # Find columns x in this row that are occupied
        occupied_x = []
        for x in range(Mx):
            k = ((My * z) + y) * Mx + x
            if D[k]:
                occupied_x.append(x)
                
        if not occupied_x:
            O[row_idx] = 0
            continue
            
        # Find smallest offset O[row_idx] such that no collision in bH
        # Start search from previous row's offset + 1 (or 0 for first row)
        start_offset = O[row_idx - 1] + 1 if row_idx > 0 else 0
        
        found = False
        offset = start_offset
        while not found:
            collision = False
            for x in occupied_x:
                if bH[offset + x]:
                    collision = True
                    break
            if not collision:
                # Mark these positions as used
                for x in occupied_x:
                    bH[offset + x] = 1
                O[row_idx] = offset
                found = True
            else:
                offset += 1
                
    # 6. Populate Hash Table H and Item List L
    # H maps hash index to index in L
    # L stores the actual data
    H = [0] * len(bH) # Or sized appropriately
    L = []
    
    # We need to map each occupied cell to a unique entry in L
    # And store the pointer in H
    # Note: The exact structure of H depends on if it stores indices into L or values directly.
    # Based on [S2, unofficial], H stores indexes into L.
    
    # Re-iterate to fill H and L
    # We need a way to assign unique IDs to items or cells.
    # Assuming each occupied cell has associated data.
    
    cell_to_L_index = {}
    L_index_counter = 0
    
    for row_idx in range(My * Mz):
        y = row_idx // Mz
        z = row_idx % Mz
        offset = O[row_idx]
        
        for x in range(Mx):
            k = ((My * z) + y) * Mx + x
            if D[k]:
                hash_idx = offset + x
                # Assign a unique index in L for this cell
                # In a real scenario, 'items' would provide the data for this cell
                # Here we simulate storing an ID or value
                data_value = get_data_for_cell(x, y, z) 
                L.append(data_value)
                cell_to_L_index[k] = L_index_counter
                
                # Store the index into L in H
                H[hash_idx] = L_index_counter
                L_index_counter += 1
                
    return L, H, O, D

def lookup_hash_grid(x, y, z, grid_dims, O, D, H, L):
    Mx, My, Mz = grid_dims
    k = ((My * z) + y) * Mx + x
    
    if not D[k]:
        return None # Empty cell
        
    row_idx = (y * Mz + z)
    offset = O[row_idx]
    hash_idx = offset + x
    
    if hash_idx >= len(H):
        return None # Out of bounds
        
    L_index = H[hash_idx]
    return L[L_index]
```

**Parameters:**
- `grid_dims` ($M_x, M_y, M_z$): Determined by a heuristic balancing memory usage and collision probability during construction [S2, S3, unofficial].
- `log2_hashmap_size`: In dynamic variants (like [S5, unofficial]), this defines the table size $T = 2^{\text{log2\_hashmap\_size}}$ [S5, unofficial].

**Edge Cases and Pitfalls:**
- **Construction Time:** The row displacement algorithm can have worst-case time complexity $O(M^{3/2})$ or worse if many rows require large displacements to find non-colliding offsets [S1].
- **Static Data:** This method is designed for static scenes. Updating the grid (adding/removing items) requires rebuilding the hash table [S2, S3, unofficial].
- **Memory Overhead:** The temporary buffer $bH$ and the domain bits $D$ add overhead. The hash table $H$ must be sized to accommodate the maximum possible displacement [S2, S3, unofficial].
- **Precision:** Coordinate mapping relies on integer arithmetic. Floating-point positions must be discretized to grid indices before hashing [S2, S3, unofficial].

## Subtopics

### [Voxel World Data Structure](1101-voxel-world-data-structure.md)
A sparse representation that stores voxel existence using a hash set of integer-based 3D vectors (`IVec3`). It solves the problem of efficiently tracking occupied grid positions in a voxel world while ensuring strict grid alignment. It builds on [Hash Grids](936-hash-grids.md) by applying hash-based lookup to discrete integer coordinates rather than continuous floating-point positions.

### [Hash Grid Loss](2093-hash-grid-loss.md)
A compression-oriented regularization term used within sparse volume representations to minimize the bit consumption of binary hash grids during encoding. It differs from standard reconstruction losses by optimizing the statistical distribution of the hash grid values themselves, encouraging a more concentrated grid structure that yields higher compression ratios at the cost of reconstruction fidelity.

### [Dual Point Map Insertion with Linear Probing](3043-dual-point-map-insertion-with-linear-probing.md)
This method stores dual points in a hash table using linear probing to handle collisions, mapping unique point identifiers to their 3D positions via a combined 64-bit key-value pair. It builds on the hashing strategy of [Hash Grids](936-hash-grids.md) by hashing vertex positions to bucket indices [G].

### [Multi-Level Hashed Grid Construction](3402-multi-level-hashed-grid-construction.md)
A method for rapidly building an acceleration structure that combines the render-time performance of multi-level grids with the low memory footprint of hashed grids. It solves the problem of high memory consumption in classic multi-level array grids and the poor traversal performance of one-level hashed grids for large, sparse scanned scenes. It differs from [Hash Grids](936-hash-grids.md) by adding macrocells to skip empty regions during traversal, and from [Voxel World Data Structure](1101-voxel-world-data-structure.md) by using row displacement compression to eliminate empty cells entirely.

### [Universal Hash Optimization](3456-universal-hash-optimization.md)
A technique used within Sparse Voxel Octree (SVO) structures to mitigate hash collisions in the underlying hash table. It replaces standard deterministic hashing with a randomized universal hash function, ensuring that distinct non-isomorphic nodes are distributed uniformly across hash table buckets. This reduces the average number of equality checks required during lookups, thereby improving search performance compared to naive hashing methods that may cluster nodes in specific buckets.

### [SpatialHash3D](5113-spatialhash3d.md)
A spatial index structure that uses a uniform 3D hash grid to enable efficient nearest-neighbor and radius-based queries for static datasets, providing O(1) cell lookup. It is distinguished from sibling structures like KD-trees or Octrees by its simplicity and predictable performance for static data, where implementation complexity is low and radius queries are straightforward.

### [3D Coordinate Packing](5755-3d-coordinate-packing.md)
A method for encoding three-dimensional spatial coordinates into a single integer key to serve as an index in hash maps or sparse data structures. It solves the problem of efficiently mapping 3D voxel positions to a 1D bucket index for sparse volume representation. Unlike general-purpose hash functions like those in [Hash Grids](936-hash-grids.md), this method preserves spatial locality through bit-shifting, allowing for deterministic reconstruction of coordinates from the key. It differs from [Voxel World Data Structure](1101-voxel-world-data-structure.md) by focusing on the low-level bit manipulation for hashing rather than high-level world management.

### [Hash Grid with Multiple Hash Functions](7678-hash-grid-with-multiple-hash-functions.md)
This method extends the standard Hash Grid by assigning a distinct hash function to each leaf node in the spatial hierarchy, thereby reducing hash collisions that occur when different regions of space map to the same warp space coordinates. While the parent [Hash Grids](936-hash-grids.md) uses a single global hash function for all vertices, this variant conditions the hashing parameters on the leaf node index, allowing points from different leaf nodes that occupy the same voxel in the shared warp space to be stored in different buckets.

### Integer Hash Map with Chaining
A hash table implementation for integer keys using open addressing via chaining. Supports hashing 2 or 4 integer keys into a single index, with separate structures for storing associated integer or float data [S6, unofficial]. Key generation for 2 keys masks each key to 10 bits, shifts k1 left by 10, ORs with k2, and masks to 20 bits. For 4 keys, masks to 5 bits and shifts accordingly [S6, unofficial]. Lookup traverses the chain at `table[ind]` comparing keys. Insertion prepends a new node to the chain [S6, unofficial]. Pitfall: Memory leaks if the destructor is not called, as it must iterate all buckets and delete nodes [S6, unofficial].

### Hash Map Operations
CUDA implementations for hash map insertion and lookup operations, including 3D specific variants and one where the 3D index is used as the value [S7, unofficial].

## Operating characteristics
- **Time Complexity:** Construction is $O(M^{3/2})$ in the worst case for row displacement, but typically faster for sparse data [S1]. Lookup is $O(1)$ [S2, S3, unofficial].
- **Memory Complexity:** $O(N)$ where $N$ is the number of grid cells, but effectively $O(\text{non-empty cells})$ due to the perfect hash avoiding empty cell storage [S2, S3, unofficial]. The domain bits $D$ add $N$ bits overhead [S2, S3, unofficial].
- **Performance:** Offers high render-time performance due to cache-friendly linear access and no pointer chasing [S2, S3, unofficial]. Construction time can be significant for very large scenes [S1].
- **Strengths:** Low memory overhead for static scenes, fast lookup, no collision resolution during rendering [S2, S3, unofficial].
- **Limitations:** Static data only (rebuilding required for updates), construction complexity [S1][S2, unofficial].

## Background
The method was introduced in 1977 by Aho and Ullman as a compaction scheme for transition diagrams of deterministic finite automata [S1]. It was later adapted for sparse matrix compression and 3D grid hashing in computer graphics, notably by Lagae and Dutre (2008) for ray tracing acceleration structures [S1][S2, S3, unofficial]. It is used in static volume rendering and ray tracing to accelerate access to sparse voxel data [S1][S2, S3, unofficial].

## Sources
[S1] Compact, Fast and Robust Grids for Ray Tracing, https://doi.org/10.1111/j.1467-8659.2008.01262.x, fetched 2026-10-24
[S2] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28153/, fetched 2026-10-24 (unofficial)
[S3] PDF Multi-level Grid Strategies for Ray Tracing, https://pdfs.semanticscholar.org/f922/40132d0d7340ff97ac4074b0a6906fb44c44.pdf, fetched 2026-10-25 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S6] aewallin/dualcontouring, https://github.com/aewallin/dualcontouring, fetched 2026-10-26 (unofficial)
[S7] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
