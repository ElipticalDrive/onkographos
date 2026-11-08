# Hash Grids

Also called: Coordinate Hashing, Vertex Hashing for Voxel Grid, Row Displacement Compression, Perfect Hashing for Sparse Grids

## Overview
Hash Grids are a sparse data structure that maps 3D coordinates to a compact hash table using a perfect hash function, eliminating the storage overhead of empty cells in a dense voxel grid [S1][S2, S3, unofficial]. Unlike sibling methods that rely on hierarchical tree traversal (e.g., [SparseGrid](904-sparsegrid.md), [Tree-based structures](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md)) or brick-based storage ([Brick-based Hierarchical Storage](2793-brick-based-hierarchical-storage.md)), Hash Grids achieve $O(1)$ lookup time by compressing the spatial domain into a linear array via row displacement [S1][S2, unofficial]. This method is distinct from [Stack-Based Representation](1081-stack-based-representation-sbr.md) and [Pointer Compression](1770-pointer-compression.md) as it does not use pointers or stack-based indexing, but rather static offsets and domain bits [S2, unofficial]. It is primarily used for static scenes where the geometry is known beforehand, allowing for collision-free hashing [S2, S3, unofficial].

## How it works
**Inputs:** A set of static 3D items (e.g., triangles, voxels) with bounding boxes; scene bounding box.
**Outputs:** Four static arrays: $L$ (item list), $H$ (hash table), $O$ (offset table), and $D$ (domain bits).

**Data Structures:**
1.  **Array $L$ (Item List):** A 1D array of machine words containing the indexes of all cell items (e.g., triangle indices or voxel attributes) [S2, unofficial].
2.  **Array $H$ (Hash Table):** A linearized and hashed 3D array of machine word indexes into $L$. The item list size for cell $i$ is given by $H[i] - H[i+1]$ [S2, unofficial].
3.  **Array $O$ (Offset Table):** A linearized 2D array of machine word indexes into $H$. Its size is $M_y \times M_z$, where $M_y$ and $M_z$ are the grid dimensions in the y and z directions [S2, unofficial].
4.  **Array $D$ (Domain Bits):** A bit array of size $N = M_x \times M_y \times M_z$. $D[i]$ is 1 if cell $i$ is not empty, 0 otherwise [S2, S3, unofficial].

**Grid Indexing:**
3D coordinates $(x, y, z)$ are mapped to a 1D index in $D$ using:
$$d(x, y, z) = ((M_y \times z) + y) \times M_x) + x$$
where $M_x, M_y, M_z$ are the grid dimensions [S2, S3, unofficial].

**Construction Algorithm (Row Displacement Compression):**
This method computes a perfect hash function with no collisions for static data [S1][S2, unofficial].

1.  **Bounding Box & Dimensions:** Compute the scene's bounding box. Determine grid dimensions $M_x, M_y, M_z$ using a grid heuristic [S2, S3, unofficial].
2.  **Domain Bits Initialization:** Allocate Array $D$ of size $N = M_x \times M_y \times M_z$. Initialize all bits to zero [S2, S3, unofficial].
3.  **Mark Non-Empty Cells:** For each item, if its bounding box intersects cell $i$, set $D[d(x, y, z)]$ to 1 [S2, S3, unofficial].
4.  **Offset Table Allocation:** Allocate Array $O$ of size $M_y \times M_z$. Allocate a temporary bit array $b_H$ of size $2 \times (\text{number of non-empty cells})$, initialized to zero [S2, unofficial].
5.  **Row Displacement:**
    *   For each grid row $(y, z)$ in $O$:
        *   Find the smallest offset $O[y, z]$ starting from the previous row's offset (or 0 for the first row) such that non-zero elements do not overlap [S1][S2, unofficial].
        *   This is determined by checking collisions in the row $D[d(0, y, z) \dots d(M_x-1, y, z)]$ against the temporary array $b_H$ [S2, unofficial].
        *   Record the offset in $O[y, z]$ [S1][S2, unofficial].
6.  **Hash Table Population:**
    *   For each non-zero element $C(i, j)$ in the conceptual sparse matrix (representing items in cell $(i, j)$):
        *   Map to hash table position $H[O[i] + j]$ [S1].
        *   Store the index into $L$ at this position [S2, unofficial].
    *   *Note:* The hash function is $h(i, j) = O[i] + j$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_hash_grid(items, grid_dims):
    Mx, My, Mz = grid_dims
    N = Mx * My * Mz
    
    # 1. Allocate D (Domain Bits)
    D = [0] * N
    
    # 2. Mark non-empty cells
    for item in items:
        # Determine cells intersected by item's bounding box
        cells = get_intersected_cells(item.bbox, grid_dims)
        for (x, y, z) in cells:
            idx = ((My * z) + y) * Mx + x
            D[idx] = 1
            
    # 3. Count non-empty cells for temporary array size
    num_non_empty = sum(D)
    bH = [0] * (2 * num_non_empty) # Temporary bit array
    
    # 4. Allocate O (Offset Table)
    O = [[0] * Mz for _ in range(My)]
    
    # 5. Row Displacement to fill O
    for y in range(My):
        for z in range(Mz):
            # Find smallest offset for this row
            # Start search from previous offset (simplified logic)
            offset = 0
            if y > 0:
                offset = O[y-1][z] # Assuming row-major or similar dependency
            
            # Check for collisions in the current row's domain bits
            # This part requires iterating through the row's cells and checking
            # if the target positions in H (conceptually) are occupied in bH
            # The source text implies a collision check logic here [S2, unofficial]
            while True:
                collision = False
                for x in range(Mx):
                    if D[((My * z) + y) * Mx + x] == 1:
                        # Conceptual hash position for this cell in this row
                        # In a real implementation, bH tracks occupied slots in H
                        # relative to the current offset
                        pass 
                if not collision:
                    break
                offset += 1
            O[y][z] = offset
            
    # 6. Populate H and L
    # H is indexed by O[y][z] + x
    # L stores the actual item data
    H = [0] * (N + 1) # Size N+1 to handle item list boundaries
    L = []
    
    for y in range(My):
        for z in range(Mz):
            offset = O[y][z]
            for x in range(Mx):
                idx = ((My * z) + y) * Mx + x
                if D[idx] == 1:
                    # Store index into L in H
                    # The exact mapping of items to H depends on the item list structure
                    # H[i] - H[i+1] gives the count of items in cell i
                    pass 
                    
    return L, H, O, D
```

**Alternative Hashing (Dynamic/Neural):**
For dynamic or neural rendering contexts (e.g., Instant-NGP variants), a multiplicative hash with prime constants is used [S4, S5, unofficial]:
$$\text{hash} = ((pos.x \times 73856093) \oplus (pos.y \times 19349663) \oplus (pos.z \times 83492791)) \pmod n$$
where $n$ is the hash table size [S4, unofficial]. In Python-like implementation:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hash(coords, log2_hashmap_size):
    '''
    coords: tensor of coordinates, up to 7 dimensions
    log2_hashmap_size: logarithm of T w.r.t 2
    '''
    primes = [1, 2654435761, 805459861, 3674653429, 2097192037, 1434869437, 2165219737]
    # Implementation details for XOR folding and modulo are inferred from standard practices [S5, unofficial]
    # Full logic not provided in source [S5, unofficial]
    pass
```

**Multi-Hash Variant (Leaf-Dependent):**
To alleviate conflicts in neural rendering, different hash functions can be used for different leaf nodes [S9]:
$$\text{Hash}_i(\hat{\mathbf{v}}) = \left( \bigoplus_{k=1}^3 (\hat{\mathbf{v}}_k \times \pi_{i,k} + \Delta_{i,k}) \right) \bmod L$$
where $\pi_{i,k}$ and $\Delta_{i,k}$ are random large primes fixed for leaf node $i$, and $L$ is the hash table length [S9].

## Subtopics

### [Voxel World Data Structure](1101-voxel-world-data-structure.md)
A sparse representation using a hash set of integer-based 3D vectors (`IVec3`) to track occupied grid positions [1101-voxel-world-data-structure.md]. It differs from siblings by using simple existence checks rather than storing attributes or bricks [1101-voxel-world-data-structure.md]. It builds on Hash Grids by applying hash-based lookup to discrete integer coordinates [1101-voxel-world-data-structure.md].

### [Hash Grid Loss](2093-hash-grid-loss.md)
A regularization term in compressive neural volume rendering that minimizes the bit consumption of the binary occupancy structure during training [2093-hash-grid-loss.md]. It models the probability distribution of binary hash grid values to drive a more concentrated structure [2093-hash-grid-loss.md]. It is distinct from spatial indexing methods like Voxel World Data Structure [2093-hash-grid-loss.md].

### [Dual Point Map Insertion with Linear Probing](3043-dual-point-map-insertion-with-linear-probing.md)
Stores dual points in a hash table using linear probing for collision handling, mapping unique point IDs to 3D positions via a 64-bit key-value pair [3043-dual-point-map-insertion-with-linear-probing.md]. It builds on Hash Grids by hashing vertex positions to bucket indices [3043-dual-point-map-insertion-with-linear-probing.md].

### [Multi-Level Hashed Grid Construction](3402-multi-level-hashed-grid-construction.md)
Combines the low memory footprint of hashed grids with the traversal efficiency of multi-level grids using macrocells to skip empty regions [3402-multi-level-hashed-grid-construction.md]. It provides approximately twice the render time performance for large scanned scenes compared to one-level hashed grids [3402-multi-level-hashed-grid-construction.md]. It builds on Hash Grids by adding hierarchical resolution levels [3402-multi-level-hashed-grid-construction.md].

### [Universal Hash Optimization](3456-universal-hash-optimization.md)
Mitigates hash collisions in Sparse Voxel Octree (SVO) structures by replacing deterministic hashing with randomized universal hash functions [3456-universal-hash-optimization.md]. This ensures uniform distribution of non-isomorphic nodes, improving search performance [3456-universal-hash-optimization.md].

### SpatialHash3D
A spatial index using a uniform 3D hash grid for efficient nearest-neighbor and radius-based queries on static datasets, providing $O(1)$ cell lookup [5113-spatialhash3d.md]. It prioritizes predictable performance for static interpolation tasks, differing from dynamic voxel world structures [5113-spatialhash3d.md].

### Integer Hash Map with Chaining
A hash table for integer keys using open addressing via chaining, supporting 2 or 4 integer keys mapped to a single index [S6, unofficial]. Key generation masks and shifts keys (e.g., 10 bits for 2 keys, 5 bits for 4 keys) into a 20-bit index [S6, unofficial]. It uses separate structures for integer/float data and requires a destructor to prevent memory leaks [S6, unofficial].

### 3D Coordinate Packing
Packs three 16-bit voxel coordinates $(x, y, z)$ into a single 64-bit integer key for hash map indexing using a union struct [S7, unofficial]. The key is formed as `key = (z << 32) | (y << 16) | x` (assuming little-endian) [S7, unofficial]. It is limited to 16-bit signed coordinates and relies on architecture-specific alignment [S7, unofficial].

### Hash Grid with Multiple Hash Functions
Uses different hash functions for different leaf nodes in neural rendering to reduce conflicts [S9]. It maps points to warp space, hashes neighboring vertices using leaf-specific primes and offsets, and trilinearly interpolates feature vectors before feeding them to an MLP [S9].

## Operating characteristics
**Time Complexity:** Construction is $O(M^{3/2})$ in the worst case for row displacement compression [S1]. Lookup is $O(1)$ [S2, S3, unofficial].
**Memory Complexity:** Proportional to the number of non-empty cells plus overhead for domain bits and offsets. It avoids storing empty cells, making it memory-efficient for sparse volumes [S1][S2, unofficial].
**Performance:** Tested on scenes like "Buddha" (1.09M triangles) and "Dragon" (871.41K triangles), showing competitive memory usage (e.g., 18.67 MB for Buddha) and adequate render times [S2, S3, unofficial].
**Strengths:** Constant-time lookup, low memory footprint for sparse data, no pointer overhead [S1][S2, unofficial].
**Limitations:** Static data requirement for perfect hashing (dynamic updates require rehashing or chaining variants) [S1][S2, unofficial]. Collision handling in dynamic variants can degrade performance [S6, unofficial].

## Background
Introduced in 1977 by Aho and Ullman for compaction of deterministic finite automata transition diagrams [S1]. Adapted for ray tracing by Lagae and Dutre (2008) to compress sparse 3D matrices and avoid storing empty array cells [S2, S3, unofficial]. It is a foundational technique for sparse voxel grids in modern rendering, including neural radiance fields (Instant-NGP) [S5, unofficial][S9].

## Sources
[S1] Compact, Fast and Robust Grids for Ray Tracing, https://doi.org/10.1111/j.1467-8659.2008.01262.x, fetched 2026-10-24
[S2] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28153/, fetched 2026-10-24 (unofficial)
[S3] PDF Multi-level Grid Strategies for Ray Tracing, https://pdfs.semanticscholar.org/f922/40132d0d7340ff97ac4074b0a6906fb44c44.pdf, fetched 2026-10-25 (unofficial)
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S6] aewallin/dualcontouring, https://github.com/aewallin/dualcontouring, fetched 2026-10-26 (unofficial)
[S7] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S8] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S9] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
