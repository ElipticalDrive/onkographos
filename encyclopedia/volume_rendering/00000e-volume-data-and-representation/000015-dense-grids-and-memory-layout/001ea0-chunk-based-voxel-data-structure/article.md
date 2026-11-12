# Chunk-based Voxel Data Structure

Also called: Voxel Chunking, Chunked Voxel World, Block chunking, Chunk-Based Block Storage, Voxel Chunk Data Structure, Chunked Voxel World Storage, Block Storage

## Overview
The Chunk-based Voxel Data Structure manages large-scale or infinite 3D voxel worlds by partitioning the continuous domain into fixed-size 3D blocks (chunks), storing voxel data in flat arrays within each chunk, and tracking active chunks via a map or list [S1, S3, S4, unofficial]. It solves the memory inefficiency and update latency of monolithic dense grids by enabling spatial locality, incremental mesh regeneration, and efficient serialization [S2, S3, S6, unofficial]. Unlike [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) which uses hierarchical pointers to skip empty space, this method relies on fixed-size dense grids within chunks, and differs from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by explicitly managing the coordinate translation logic for chunk boundaries [S1, S2, S4, unofficial].

## How it works
**Inputs:** Global voxel coordinates $(x, y, z)$, chunk dimensions $(C_x, C_y, C_z)$, and voxel data (e.g., material ID, density).
**Outputs:** A structured collection of chunks, each containing a flat array of voxel data and metadata (mesh references, active flags).

**Data Structures:**
1.  **Chunk:** A container holding:
    *   `data`: A flat 1D array (e.g., `Uint8Array`, `uint8_t`) of size $C_x \times C_y \times C_z$ [S3, S5, S7, unofficial].
    *   `mesh`: Reference to the generated geometry (e.g., OpenGL VBO, GPU buffer) [S5, S7, unofficial].
    *   `coords`: Global chunk coordinates $(c_x, c_y, c_z)$ [S5, unofficial].
2.  **World/Manager:** A spatial hash or array mapping chunk coordinates to Chunk instances [S1, S4, unofficial].
    *   In some implementations, a `chunkIdMap` tracks active chunk IDs to avoid sparse allocation overhead [S1, unofficial].
    *   Active block tracking may use a bitmask per column (e.g., `chunksActive[chunkId][x][y]` as a 16-bit integer for Z-coordinates 0-15) [S1, unofficial].

**Indexing and Coordinate Conversion:**
Global coordinates are mapped to chunk indices and local coordinates using integer division and modulo (or bit shifts for power-of-two sizes) [S2, S3, unofficial].

1.  **Chunk Index:**
    $$ c_x = \lfloor x / C_x \rfloor, \quad c_y = \lfloor y / C_y \rfloor, \quad c_z = \lfloor z / C_z \rfloor $$
    For power-of-two $C$, this is equivalent to right-shift: $c_x = x \gg \log_2(C_x)$ [S2, unofficial].

2.  **Local Coordinates:**
    $$ l_x = x \pmod{C_x}, \quad l_y = y \pmod{C_y}, \quad l_z = z \pmod{C_z} $$
    For power-of-two $C$, this is equivalent to bitwise AND: $l_x = x \ \& \ (C_x - 1)$ [S2, unofficial].

3.  **Flat Array Index:**
    The local coordinates are mapped to a 1D index in the chunk's `data` array. The standard row-major layout is:
    $$ I = l_x + l_y \cdot C_x + l_z \cdot C_x \cdot C_y $$
    *Note: Some implementations use different strides, e.g., $I = l_y \cdot C_x \cdot C_z + l_z \cdot C_x + l_x$ [S5, unofficial].*

4.  **Chunk ID Generation:**
    Chunks are uniquely identified by their global coordinates. A common scheme encodes $(c_x, c_y, c_z)$ into a string or hash:
    $$ \text{ID} = \text{encode}(c_x, c_y, c_z) $$
    In one implementation, an offset is calculated as $\text{offset} = \text{blockSize} \times \text{chunkSize}$, and the ID is a base64-encoded string of `"cx,cy,cz"` [S1, unofficial].

**Algorithm: Add/Update Voxel**
1.  Compute chunk coordinates $(c_x, c_y, c_z)$ and local coordinates $(l_x, l_y, l_z)$ from global $(x, y, z)$ [S1, S2, unofficial].
2.  Retrieve or initialize the Chunk at $(c_x, c_y, c_z)$ from the World manager [S1, S2, unofficial].
    *   If the chunk does not exist, create a new Chunk with empty `data` and add it to the manager [S1, unofficial].
3.  Compute the flat index $I$ within the chunk's `data` array [S3, unofficial].
4.  Set `data[I]` to the new voxel value (e.g., material ID) [S3, unofficial].
5.  Mark the chunk as modified/dirty to trigger mesh regeneration [S8, unofficial].
6.  If the modification is at a chunk boundary, mark adjacent chunks as modified to handle cross-chunk faces [S8, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Chunk:
    def __init__(self, cx, cy, cz, size_x, size_y, size_z):
        self.cx = cx
        self.cy = cy
        self.cz = cz
        self.size_x = size_x
        self.size_y = size_y
        self.size_z = size_z
        # Flat array for voxel data (e.g., material IDs)
        self.data = [0] * (size_x * size_y * size_z)
        self.mesh = None
        self.dirty = True

    def get_index(self, lx, ly, lz):
        # Row-major indexing: x is fastest varying
        return lx + ly * self.size_x + lz * self.size_x * self.size_y

    def set_voxel(self, lx, ly, lz, value):
        idx = self.get_index(lx, ly, lz)
        self.data[idx] = value
        self.dirty = True

class World:
    def __init__(self, chunk_size_x=32, chunk_size_y=32, chunk_size_z=32):
        self.chunk_size_x = chunk_size_x
        self.chunk_size_y = chunk_size_y
        self.chunk_size_z = chunk_size_z
        # Map from chunk coordinates (tuple) to Chunk instance
        self.chunks = {}

    def get_chunk_coords(self, x, y, z):
        cx = x // self.chunk_size_x
        cy = y // self.chunk_size_y
        cz = z // self.chunk_size_z
        return (cx, cy, cz)

    def get_local_coords(self, x, y, z):
        lx = x % self.chunk_size_x
        ly = y % self.chunk_size_y
        lz = z % self.chunk_size_z
        return (lx, ly, lz)

    def set_voxel(self, x, y, z, value):
        # Handle negative coordinates if necessary (e.g., two's complement or offset)
        # Assuming non-negative for simplicity, or using floor division for negatives
        cx, cy, cz = self.get_chunk_coords(x, y, z)
        lx, ly, lz = self.get_local_coords(x, y, z)
        
        chunk_key = (cx, cy, cz)
        if chunk_key not in self.chunks:
            self.chunks[chunk_key] = Chunk(cx, cy, cz, self.chunk_size_x, self.chunk_size_y, self.chunk_size_z)
        
        chunk = self.chunks[chunk_key]
        chunk.set_voxel(lx, ly, lz, value)

    def get_voxel(self, x, y, z):
        cx, cy, cz = self.get_chunk_coords(x, y, z)
        lx, ly, lz = self.get_local_coords(x, y, z)
        
        chunk_key = (cx, cy, cz)
        if chunk_key not in self.chunks:
            return 0 # Air
        
        chunk = self.chunks[chunk_key]
        idx = chunk.get_index(lx, ly, lz)
        return chunk.data[idx]
```

**Parameters:**
*   **Chunk Size:** Commonly $16^3$, $32^3$, or non-cubic like $16 \times 256 \times 16$ [S5, S6, unofficial].
    *   $16^3$ or $32^3$ are typical for balanced remeshing and triangle counts [S6, unofficial].
    *   $16 \times 256 \times 16$ keeps data small and re-meshing fast for tall, thin worlds [S5, unofficial].
*   **Voxel Data Type:** 8-bit unsigned integer (material ID) [S3, S5, unofficial], or 16-bit float (density) + 8-bit material ID [G].
*   **Boundary Policy:** Out-of-bounds queries return a default value (e.g., 0 for Air) [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Duplicate Detection:** If storing block objects rather than IDs, reference equality (`chunk[i] == block`) may fail for distinct instances with identical coordinates [S1, unofficial].
*   **Memory Management:** Copying Chunk instances by value (e.g., in C++ `std::vector` iteration) can cause memory disasters due to large data arrays and associated GPU resources; use smart pointers or references [S7, unofficial].
*   **Boundary Handling:** Modifications at chunk edges require notifying adjacent chunks to regenerate their shared faces [S8, unofficial].
*   **Negative Coordinates:** Modulo operations on negative numbers behave differently in different languages (e.g., Python vs. C++/JavaScript). Ensure consistent floor-based division for chunk indexing [G].

## Subtopics

### [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md)
A Sparse Voxel Octree (SVO) is a hierarchical data structure that represents the geometry of a 3D voxelized scene by recursively decomposing space only where active voxels exist, effectively eliminating empty regions. It solves the memory inefficiency of dense grids by storing only the necessary nodes, enabling efficient ray-traced rendering and Level-of-Detail (LoD) management. Unlike flat grid structures, it provides tight compression of sparse spaces, and unlike KD-trees, it offers uniform subdivision at every level, ensuring smoothly-degrading LoD and consistent processing cost bounds. It differs from its sibling [Sparse Voxel Octree with Bricks](2354-sparse-voxel-octree-with-bricks.md) by focusing on a compact, pointer-based node structure optimized for direct shader traversal rather than grouped bricks for cache locality.

### [Terrain Chunk Data Structure](2420-terrain-chunk-data-structure.md)
The Terrain Chunk Data Structure represents a height-field-based terrain as a hierarchical collection of triangular patches, enabling efficient multiresolution rendering and compression. Unlike the parent [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md) which manages discrete voxel IDs, this method manages continuous geometric surfaces (triangles) grouped into chunks for batched rendering, cache locality, and adaptive refinement. It is distinct from [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by focusing on surface meshes rather than volumetric occupancy, and from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by using a bintree hierarchy of patches instead of fixed 3D grids.

### [Chunk-Based World Storage](2858-chunk-based-world-storage.md)
Chunk-Based World Storage maps global voxel coordinates to specific chunks and local voxel indices to enable efficient storage and retrieval in a spatially partitioned voxel world. It solves the problem of managing infinite or large-scale voxel data by dividing the world into fixed-size 3D blocks, optimizing memory locality and access patterns. Unlike [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) which uses hierarchical pointers, this method relies on fixed-size dense grids within chunks, and differs from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by explicitly managing the coordinate translation logic for chunk boundaries.

### [Chunk Border Neighbor Access](2860-chunk-border-neighbor-access.md)
Chunk Border Neighbor Access is a strategy for handling neighbor lookups at chunk boundaries in chunk-based voxel data structures by providing a read-only neighborhood view trait. It solves the problem of distinguishing between internal voxels and boundary voxels during operations like meshing or rendering, treating chunk borders as semantically identical to internal voxels. This approach separates the logic of neighbor retrieval from the logic of neighbor occlusion or processing, allowing the meshing or rendering code to query local and cross-chunk neighbors through a uniform interface. It differs from sibling methods like [Padded Chunk Data Structure](4918-padded-chunk-data-structure.md) by avoiding memory duplication of boundary voxels, and from [Chunk Neighbor Retrieval](6783-chunk-neighbor-retrieval.md) by providing a direct data access interface rather than just coordinate offsets.

### [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md)
Compact Voxel Section Storage is a memory-optimized representation for a fixed-size $16 \times 16 \times 16$ voxel section (4096 voxels) that minimizes heap allocation by distinguishing between empty, uniform, and sparse material distributions. Unlike dense arrays which always consume fixed memory, this method uses bit-packed palette indices for mixed materials, achieving $O(1)$ bytes for trivial sections and compact storage for low-entropy terrain, while supporting $O(1)$ random access.

### [Voxel Chunk Data Management and Residency](6751-voxel-chunk-data-management-and-residency.md)
This method manages the memory residency of voxel chunks in an infinite voxel world by tracking state transitions between raw, uniform, and compressed forms to reduce memory footprint for distant chunks. It solves the problem of unbounded memory usage in dense voxel grids by converting raw voxel arrays into compact representations (Uniform or Compressed) and dropping the resident raw array, while maintaining data consistency through versioning and mutation flags. Unlike sibling structures that focus on spatial partitioning or octree traversal, this method operates on the data lifecycle within a fixed-size chunk, using distance-keyed asynchronous compression to prioritize memory savings for far chunks.

### [Clustered Grid Cell (CGC) Data Structure](9217-clustered-grid-cell-cgc-data-structure.md)
The Clustered Grid Cell (CGC) data structure is a memory-efficient representation for isosurface rendering that stores active grid cells directly as rendering primitives, eliminating the need for the original volume scalar data during the rendering phase. It sets itself apart from previous cell rasterization approaches, such as Liu et al.'s method, by storing eight corner scalar values with each cell, allowing full rendering without GPU-resident volume textures, while achieving approximately 37% of the memory usage of typical efficient mesh-based representations.

### Chunk Data Modification & Queue Management
Processes chunk modifications, expands voxel arrays if sparse, updates block types, tracks adjacent chunks for mesh regeneration, and queues affected chunks for mesh loading [S8, unofficial]. When a voxel is modified, if the voxel array length is 1, it is expanded to full chunk size by duplicating the single block data. The block type is updated at the calculated index. If the modification is at a chunk edge, adjacent chunks are added to a set for mesh regeneration, and both the current and adjacent chunks are queued for mesh loading [S8, unofficial].

### Voxel Layer System
A data structure system allowing non-destructive changes to voxel worlds by storing voxels in separate VoxelLayer structures rather than directly in the chunk array, enabling toggling visibility of different content sets (e.g., terrain vs. castle) [S9, unofficial]. Voxels are assigned to specific layers (e.g., background for terrain, top for castles). The system supports toggling visibility of specific layers without modifying the underlying voxel data. Default implementations often use two layers (background and top), though the system supports more [S9, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Voxel Access: $O(1)$ for chunk lookup and local indexing [S1, S3, unofficial].
    *   Mesh Generation: $O(N)$ where $N$ is the number of voxels in the chunk, but only for modified chunks [S6, unofficial].
*   **Memory Complexity:**
    *   Per Chunk: $O(C_x \cdot C_y \cdot C_z)$ for the flat array [S3, S5, unofficial].
    *   Total: $O(K \cdot C_x \cdot C_y \cdot C_z)$ where $K$ is the number of active chunks [S1, unofficial].
*   **Measured Performance:**
    *   Chunk sizes of $16^3$ or $32^3$ balance remeshing speed and triangle count [S6, unofficial].
    *   Non-cubic chunks like $16 \times 256 \times 16$ can optimize for specific world shapes (e.g., tall terrain) [S5, unofficial].
*   **Strengths:**
    *   Efficient spatial locality for rendering and updates [S2, unofficial].
    *   Incremental updates allow partial mesh regeneration [S6, unofficial].
    *   Simplifies world serialization by saving chunks independently [S6, unofficial].
*   **Limitations:**
    *   Inefficient for highly sparse worlds compared to octrees [S2, unofficial].
    *   Fixed chunk size may lead to memory waste if chunks are mostly empty [S7, unofficial].
    *   Boundary handling requires careful neighbor management [S8, unofficial].
*   **When to Use:** Large-scale voxel worlds with frequent local updates, where spatial locality and incremental meshing are critical.
*   **When Not to Use:** Highly sparse scenes where memory efficiency is paramount (use SVO), or when global coherence is needed without chunk boundaries.

## Background
The Chunk-based Voxel Data Structure originated as a solution to manage the memory and processing locality of infinite voxel worlds in games like Minecraft [S5, S6, unofficial]. It was popularized by the need to efficiently mesh and render large voxel environments without regenerating the entire world mesh on every block change [S6, unofficial]. Early implementations in WebGL and JavaScript demonstrated the feasibility of chunk-based storage for real-time rendering [S1, unofficial]. Subsequent developments in C++ and Unity environments refined the structure for performance, adding support for sparse data, layer systems, and efficient memory management [S2, S4, S7, S9, unofficial].

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
[S3] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S4] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S5] Build a 3D Voxel Engine in WebGL | Tutorials, https://www.mysimulator.uk/content/tutorials/voxel-engine.html, fetched 2026-10-27 (unofficial)
[S6] kotychok/mutiny, https://github.com/kotychok/mutiny, fetched 2026-10-27 (unofficial)
[S7] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
[S8] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S9] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
