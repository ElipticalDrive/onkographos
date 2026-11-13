# Hash-Map Voxel Storage

Also called: Sparse Voxel Storage, VoxelData

## Overview
Hash-Map Voxel Storage is a sparse data structure that maps 3D voxel coordinates to their attributes (such as color or density) using a hash map, enabling efficient storage of non-empty voxels in large volumes. It solves the memory inefficiency of dense grids by only storing occupied cells, while maintaining grid alignment through integer-based coordinate keys. It differs from its parent [Voxel World Data Structure](1101-voxel-world-data-structure.md) by explicitly tracking grid bounds and packing attribute data into the hash value, and from siblings like [Multi-type Voxel Storage](1104-multi-type-voxel-storage.md) by focusing on a single unified attribute type per storage instance.

## How it works
**Inputs:** A stream or list of voxel definitions, where each definition contains a 3D integer coordinate $(x, y, z)$ and an attribute value (e.g., color).
**Outputs:** A `VoxelData` structure containing a hash map of packed attributes, a count of stored voxels, and the bounding box dimensions.

**Data Structures:**
The core structure `VoxelData` consists of:
- `datas`: A hash map (dictionary) mapping a packed 3D coordinate key to a 32-bit integer attribute value.
- `count`: An integer representing the total number of voxels stored in `datas`.
- `width`, `height`, `depth`: Integers tracking the maximum extent of the grid along each axis. These define the bounding box of the stored data.

**Coordinate Packing:**
To serve as a hash key, the 3D integer coordinate $(x, y, z)$ is packed into a single 64-bit integer (or a suitable hash key type). While the exact bit-packing scheme for the coordinate key is not explicitly detailed in the primary source [S1, unofficial], standard practice for such structures involves shifting the components to avoid collision. For implementation, assume a standard packing function `pack_key(x, y, z)` that combines the three integers.

**Attribute Packing:**
The voxel attribute (specifically color in the cited implementation) is packed into a 32-bit integer. The layout is defined as:
- Bits 0-7: Blue (B)
- Bits 8-15: Green (G)
- Bits 16-23: Red (R)
- Bits 24-31: Alpha (A)
This is stored as a single 32-bit integer value in the hash map.

**Algorithm Steps:**
1. Initialize `VoxelData` with empty `datas` map, `count` = 0, and `width` = `height` = `depth` = 0.
2. Iterate through each incoming voxel definition $(x, y, z, \text{color})$.
3. Compute the packed coordinate key $K = \text{pack\_key}(x, y, z)$.
4. Pack the color attribute into a 32-bit integer $V = (A \ll 24) | (R \ll 16) | (G \ll 8) | B$.
5. Insert or update the entry in `datas` such that `datas[K] = V`.
6. Update the bounding box dimensions:
   - `width = max(width, x + 1)`
   - `height = max(height, y + 1)`
   - `depth = max(depth, z + 1)`
7. Increment `count` by 1 for each unique voxel processed. (Note: If duplicate coordinates are possible, the count should reflect the number of entries in the hash map, not the number of input items, to maintain consistency with `datas.size()`).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelData:
    def __init__(self):
        self.datas = {}  # Hash map: packed_key -> packed_color_32bit
        self.count = 0
        self.width = 0
        self.height = 0
        self.depth = 0

    def pack_key(self, x, y, z):
        # Implementation detail: Combine x, y, z into a unique hash key.
        # A simple linear combination assuming reasonable coordinate ranges:
        # Using a large multiplier to prevent overlap.
        # Note: The source [S1, unofficial] does not specify the exact bit-shift logic for the key.
        # A robust implementation would use a standard hash or bit-packing.
        return hash((x, y, z)) # [G] Placeholder for specific key packing logic

    def pack_color(self, r, g, b, a):
        # Pack into 32-bit integer: BGR + Alpha
        # Assuming 8-bit channels
        return (a << 24) | (r << 16) | (g << 8) | b

    def add_voxel(self, x, y, z, r, g, b, a):
        key = self.pack_key(x, y, z)
        value = self.pack_color(r, g, b, a)
        
        # If the key already exists, the value is overwritten.
        # The count should reflect the number of unique keys in the map.
        if key not in self.datas:
            self.count += 1
            
        self.datas[key] = value
        
        # Update bounding box
        if x + 1 > self.width:
            self.width = x + 1
        if y + 1 > self.height:
            self.height = y + 1
        if z + 1 > self.depth:
            self.depth = z + 1

    def get_voxel(self, x, y, z):
        key = self.pack_key(x, y, z)
        if key in self.datas:
            return self.datas[key]
        return None # Or a default empty value
```

**Edge Cases and Pitfalls:**
- **Duplicate Coordinates:** If the input contains multiple voxels at the same $(x, y, z)$, the hash map will overwrite the previous value. The `count` must track the number of *unique* keys, not the number of insertions, to remain consistent with `len(datas)`.
- **Coordinate Range:** The packing function for the key must handle the expected range of coordinates. If coordinates are large, a simple linear combination may collide; a cryptographic hash or structured bit-packing is required.
- **Bounding Box Calculation:** The width/height/depth are calculated as `max_coord + 1`, implying a 1-based or exclusive upper bound interpretation. This must be consistent with how the grid is sampled during rendering.

## Subtopics

### [Voxel Table Registry](10960-voxel-table-registry.md)
The Voxel Table Registry is a centralized lookup structure that provides $O(1)$ access to voxel definitions by both unique identifier and internal name. It solves the problem of redundant disk I/O and inconsistent state by maintaining a static, in-memory map of voxel types, serving as the authoritative source for chunk generation and rendering logic. It differs from sibling hash-grid storage methods by focusing on semantic definition management rather than spatial coordinate mapping.

### [Voxel Hash Chunk Map](12207-voxel-hash-chunk-map.md)
The Voxel Hash Chunk Map (voxlvoxl) is a sparse 3D point cloud representation that partitions space into independent, hash-addressed cubic chunks to mitigate memory overhead for large-scale maps. Unlike sibling structures like Rolling Bonxai that enforce a fixed 3x3x3 active window, voxlvoxl uses a hash function for chunk indexing, allowing unlimited degrees of freedom for dynamic summoning and dismissing of regions.

### [Voxel Hash for Gaussian Splat Indexing](12601-voxel-hash-for-gaussian-splat-indexing.md)
Voxel Hash is a sparse data structure that maps each 3D Gaussian Splat to the voxel cell containing its center $\mu$, enabling efficient lookup of splats within specific voxel regions for operations like carving or stacking. It differs from sibling structures by maintaining a direct bidirectional mapping between splat indices and voxel keys, allowing for rapid identification of splats to remove or modify within a target voxel.

### Online Random Access
This variant, specifically the `[Hash, 5, 4, NN(3)]` configuration from NeuralVDB, supports random access to values directly from in-memory data without full reconstruction, balancing memory footprint and access time for real-time applications [S2]. It uses standard accelerated random access techniques (level {3, 2, 1}) to determine if a query point maps to a tile or a voxel (level 0). If the query maps to a tile, it returns the value explicitly encoded in the standard VDB structure. If it maps to a voxel (level 0), it predicts the voxel value using a regressor. This configuration has the same performance characteristics as a standard VDB tree, except leaf values require additional regression. Its in-memory footprint is an order of magnitude smaller than `[Hash, 5, 4, 3]` and it can be trivially reconstructed from `[Hash, 5, NN(4), NN(3)]` by leaving the voxel regressor unchanged, acting as a pre-cached representation [S2].

## Operating characteristics
- **Time Complexity:** Lookup is $O(1)$ on average for hash map operations. Bounding box updates are $O(1)$ per voxel.
- **Memory Complexity:** Proportional to the number of non-empty voxels. The source [S1, unofficial] does not specify the exact memory overhead of the hash map implementation, but it is significantly less than a dense grid for sparse data.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Efficient storage for sparse volumes; simple implementation; fast random access to individual voxels.
- **Limitations:** Hash map overhead (load factor, collisions); bounding box calculation requires iterating all voxels; no inherent spatial coherence for traversal (unlike tree-based structures).
- **When to use:** When the volume is sparse and random access to specific voxel attributes is required.
- **When not to use:** When the volume is dense (use dense arrays) or when hierarchical traversal is needed for rendering (use tree-based structures like VDB or Octrees).

## Background
The structure is described in the context of voxel world data management, specifically in the repository `nustxujun/AfterHumanDeclined` [S1, unofficial]. It serves as a foundational sparse storage mechanism for voxel-based applications. The NeuralVDB paper [S2] discusses related hash-based sparse volume representations, particularly in the context of neural network regression for voxel values, highlighting the trade-offs between memory footprint and access speed.

## Sources
[S1] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S2] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.emergentmind.com/papers/2208.04448, fetched 2026-10-27
