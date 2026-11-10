# Hash-Map Voxel Storage

Also called: Sparse Voxel Storage, VoxelData

## Overview
Hash-Map Voxel Storage is a sparse voxel representation that maps 3D integer coordinates to voxel attributes (such as color or density) using a hash table, building on the grid-alignment guarantees of [Voxel World Data Structure](1101-voxel-world-data-structure.md). It differs from sibling structures like [Multi-type Voxel Storage](1104-multi-type-voxel-storage.md) by focusing on a single unified attribute map per coordinate, and from [Brick-based Sparse Voxel Lookup with Open-Addressing Hash](2587-brick-based-sparse-voxel-lookup-with-open-addressing-hash.md) by storing individual voxels rather than compressed brick tiles. This method solves the problem of efficiently storing large, sparse volumetric datasets where most of the space is empty, avoiding the memory overhead of dense 3D arrays.

## How it works
The method stores voxel existence and attributes in a hash map, indexed by packed 3D coordinates. It tracks the bounding box of the stored data to facilitate coordinate normalization or rendering bounds.

### Inputs and Outputs
- **Input**: A stream or list of voxel definitions, each containing a 3D integer position $(x, y, z)$ and an attribute value (e.g., color).
- **Output**: A `VoxelData` structure containing the hash map, the count of stored voxels, and the bounding box dimensions.

### Data Structures
The core data structure is `VoxelData`, which contains the following fields [S1, unofficial]:
- `datas`: A hash map (dictionary) where keys are packed 3D coordinates and values are packed voxel attributes.
- `count`: An integer representing the number of voxels currently stored in `datas`.
- `width`: An integer tracking the maximum $x$ coordinate (grid bounds).
- `height`: An integer tracking the maximum $y$ coordinate (grid bounds).
- `depth`: An integer tracking the maximum $z$ coordinate (grid bounds).

### Coordinate Packing
To use 3D coordinates as hash keys, they are packed into a single 32-bit integer. The standard layout described in the sources packs color into a 32-bit integer (BGR + alpha) [S1, unofficial]. For the coordinate key itself, a common implementation (implied by the use of `IVec3` in the parent structure and 32-bit packing) maps $(x, y, z)$ to a unique integer. While the exact bit-shift formula for the coordinate key is not explicitly detailed in the provided notes, it must be a bijective mapping from $\mathbb{Z}^3$ to $\mathbb{Z}$ that preserves locality or allows efficient hashing. The attribute value is packed as a 32-bit integer:
$$ \text{Color}_{32} = (A \ll 24) | (B \ll 16) | (G \ll 8) | R $$
where $R, G, B, A$ are 8-bit unsigned integers.

### Algorithm Steps
1. **Initialization**: Create an empty `VoxelData` instance. Initialize `count = 0`, `width = 0`, `height = 0`, `depth = 0`. Initialize `datas` as an empty hash map.
2. **Iteration**: For each incoming voxel with position $(x, y, z)$ and color $(R, G, B, A)$:
    a. **Update Bounds**: Update the bounding box dimensions:
       $$ \text{width} = \max(\text{width}, x) $$
       $$ \text{height} = \max(\text{height}, y) $$
       $$ \text{depth} = \max(\text{depth}, z) $$
    b. **Pack Attribute**: Convert the color to a 32-bit integer:
       $$ \text{val} = (A \ll 24) | (B \ll 16) | (G \ll 8) | R $$
    c. **Pack Key**: Compute the hash key $K$ from $(x, y, z)$. [G: A standard approach for 3D integer hashing is to use a spatial hash or bit-interleaving, e.g., $K = \text{hash}(x, y, z)$].
    d. **Store**: Insert or update the entry in `datas` with key $K$ and value $\text{val}$.
    e. **Increment Count**: If the key was not previously present, increment `count`.
3. **Completion**: Return the `VoxelData` structure.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelData:
    def __init__(self):
        self.datas = {}  # Hash map: int -> int
        self.count = 0
        self.width = 0
        self.height = 0
        self.depth = 0

    def add_voxel(self, x: int, y: int, z: int, r: int, g: int, b: int, a: int):
        # Update bounds
        if x > self.width:
            self.width = x
        if y > self.height:
            self.height = y
        if z > self.depth:
            self.depth = z

        # Pack color into 32-bit integer (BGR + Alpha)
        # Note: Source [S1, unofficial] specifies BGR + alpha packing.
        packed_color = (a << 24) | (b << 16) | (g << 8) | r

        # Pack coordinate into hash key
        # Implementation detail: Use a standard 3D hash function or bit interleaving
        # to map (x, y, z) to a unique integer key.
        key = hash_3d_to_int(x, y, z)

        # Store in hash map
        if key not in self.datas:
            self.count += 1
        self.datas[key] = packed_color

    def get_voxel(self, x: int, y: int, z: int):
        key = hash_3d_to_int(x, y, z)
        if key in self.datas:
            return self.datas[key]
        return None

def hash_3d_to_int(x: int, y: int, z: int) -> int:
    # [G] A common spatial hash for 3D integers.
    # This is a placeholder for the specific hashing algorithm used in the source,
    # which is not explicitly defined in the provided notes.
    # A robust implementation might use:
    # return hash((x, y, z))
    # Or a custom bit-interleaving scheme.
    return hash((x, y, z))
```

### Parameters and Edge Cases
- **Hash Collisions**: The hash map must handle collisions. Standard hash map implementations (e.g., open addressing or chaining) should be used.
- **Coordinate Range**: The bounding box variables (`width`, `height`, `depth`) track the maximum coordinate. If negative coordinates are used, the bounds logic must be adjusted to track min/max or use an offset. The sources do not specify handling for negative coordinates.
- **Precision**: The 32-bit color packing assumes 8-bit channels. If higher precision is needed, the packing scheme must be extended, but the source specifies 32-bit.

## Subtopics

### [Voxel Hash Chunk Map](12207-voxel-hash-chunk-map.md)
The Voxel Hash Chunk Map (voxlvoxl) is a sparse 3D point cloud representation that partitions space into independent, hash-addressed cubic chunks to mitigate memory overhead for large-scale maps. Unlike sibling structures like Rolling Bonxai that enforce a fixed 3x3x3 active window, voxlvoxl uses a hash function for chunk indexing, allowing unlimited degrees of freedom for dynamic summoning and dismissing of regions.

### Online Random Access
**Definition**: A variant supporting random access to values directly from in-memory NeuralVDB data without full reconstruction, balancing memory footprint and access time for real-time applications [S2].

**Steps**:
1. Use standard accelerated random access techniques (level {3, 2, 1}) to determine if a query point maps to a tile or a voxel (level 0) [S2].
2. If the query maps to a tile, return the value explicitly encoded in the standard VDB structure [S2].
3. If the query maps to a voxel (level 0), predict the voxel value using the regressor [S2].

**Parameters**: Configuration: [Hash, 5, 4, NN(3)] [S2].

**Comparison**: Performance characteristics are same as standard VDB tree except leaf values require additional regression. Memory footprint is an order of magnitude smaller than [Hash, 5, 4, 3] [S2].

### Voxel Table Registry
**Definition**: A central voxel registry providing O(1) lookup by ID and name. It manages voxel definitions, supporting loading from a ScriptableObject database or falling back to hardcoded definitions. ID 0 is reserved for air/null [S3, unofficial].

**Steps**:
1. **LoadFromDatabase**: Convert the database to runtime definitions. Create the name-to-ID map by iterating through `_runtimeVoxels`, using index $i + 1$ as the ID. Skip voxels with null/empty `InternalName` [S3, unofficial].

**Parameters**: `DATABASE_RESOURCE_PATH`: The path to the default database asset in the Resources folder. Value: "VoxelDatabase" [S3, unofficial].

**Layout**:
- `_runtimeVoxels`: A static array of VoxelDefinition objects, indexed by $(id - 1)$ [S3, unofficial].
- `_nameIdMap`: A static `Dictionary<string, uint>` mapping internal voxel names to their IDs [S3, unofficial].

## Operating characteristics
- **Time Complexity**: Insertion and lookup are $O(1)$ on average for hash map operations, assuming a good hash function and low collision rate.
- **Memory Complexity**: $O(N)$ where $N$ is the number of non-empty voxels. The memory overhead is determined by the hash map implementation and the size of the packed attributes.
- **Strengths**: Efficient storage for sparse data; simple implementation; fast random access to individual voxels.
- **Limitations**: Does not support spatial coherence optimizations like brick compression; hash collisions can degrade performance; bounding box tracking requires iteration over all voxels.
- **When to use**: When the voxel grid is very sparse and individual voxel access is required.
- **When not to use**: When the grid is dense or when spatial locality is critical for performance (consider brick-based methods).

## Background
The Hash-Map Voxel Storage is a fundamental sparse voxel representation technique. It is used in various voxel-based applications, including game development and scientific visualization, where efficient storage of sparse volumetric data is required. The specific implementation details provided in the sources are derived from unofficial repositories and research papers on neural volume representations.

## Sources
[S1] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S2] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.emergentmind.com/papers/2208.04448, fetched 2026-10-27
[S3] VektorKnight/vektor-voxels, https://github.com/VektorKnight/vektor-voxels, fetched 2026-10-27 (unofficial)
