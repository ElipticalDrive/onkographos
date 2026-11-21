# Chunk-based Voxel Data Structure

Also called: Voxel Chunking, Chunked Voxel World, Block chunking, Chunk-Based Block Storage, Voxel Chunk Data Structure, Chunked Voxel World Storage, Block Storage

## Overview
The Chunk-based Voxel Data Structure manages large-scale or infinite 3D voxel worlds by partitioning the global coordinate space into fixed-size 3D blocks called chunks [S1, S3, S5, S6, S7, unofficial]. It solves the memory inefficiency and update latency of monolithic dense grids by ensuring that spatial locality is preserved within small, manageable units, allowing for incremental mesh regeneration and efficient world serialization [S6, S7, unofficial]. Unlike its sibling [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md), which uses hierarchical pointers to skip empty space, this method relies on fixed-size dense grids within chunks, and differs from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by operating at a coarser granularity (typically 16³ to 32³) and explicitly managing the coordinate translation logic for chunk boundaries [S1, S2, S6, S8, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Global 3D voxel coordinates $(x, y, z)$ and voxel data (e.g., material ID, density, or block type).
*   **Output:** A mapped chunk identifier and a local index within that chunk's data structure, enabling read/write access to the voxel data.

**Data Structures:**
The implementation varies by language and specific optimization goals, but generally consists of:
1.  **Chunk Container:** A class or struct holding the voxel data and metadata.
    *   **Voxel Data:** A flat 1D array (e.g., `Uint8Array`, `uint8_t[]`) representing the 3D grid. The size is $S_x \times S_y \times S_z$, where $S$ are the chunk dimensions [S3, S5, S7, unofficial].
    *   **Chunk Coordinates:** Global indices $(cx, cy, cz)$ identifying the chunk's position in the world grid [S5, S7, unofficial].
    *   **Mesh Reference:** A pointer or handle to the rendered geometry (e.g., OpenGL VBO, Unity Mesh) associated with the chunk [S4, S5, S7, unofficial].
2.  **World Map:** A data structure mapping chunk coordinates to chunk containers.
    *   **Hash Map / Dictionary:** Maps a string or integer key (derived from $cx, cy, cz$) to the chunk object [S1, S5, unofficial].
    *   **Flat Array:** In some implementations, chunks are stored in a pre-allocated flat array indexed by a computed stride based on maximum world dimensions [S2, unofficial].
    *   **Active Tracking:** A bitlist or boolean array may track which chunks are active or contain non-air voxels to optimize iteration [S1, unofficial].

**Numbered Steps:**

1.  **Coordinate Conversion:** Convert global voxel coordinates $(x, y, z)$ to chunk indices $(f, g, h)$ and local coordinates $(lx, ly, lz)$.
    *   Chunk indices are typically computed using integer division or bit-shifting:
        $$ f = \lfloor x / S_x \rfloor, \quad g = \lfloor y / S_y \rfloor, \quad h = \lfloor z / S_z \rfloor $$
        If $S$ is a power of two, this is optimized to:
        $$ f = x \gg \text{ChunkShift}_x, \quad g = y \gg \text{ChunkShift}_y, \quad h = z \gg \text{ChunkShift}_z $$
        [S2, S5, unofficial]
    *   Local coordinates are computed using the modulo operator or bitwise AND:
        $$ lx = x \pmod{S_x}, \quad ly = y \pmod{S_y}, \quad lz = z \pmod{S_z} $$
        Or:
        $$ lx = x \ \& \ (S_x - 1) $$
        [S2, unofficial]

2.  **Chunk Lookup:** Determine the unique identifier for the chunk $(f, g, h)$ and retrieve or create the chunk container.
    *   **Key Generation:** The chunk key is often a string `"cx,cy,cz"` or a hash of the coordinates [S1, S5, unofficial].
    *   **Initialization:** If the chunk does not exist in the world map, create a new chunk instance, initialize its voxel data array (typically with air/0 values), and insert it into the map [S1, S2, unofficial].

3.  **Data Access:** Access or modify the voxel data using the local coordinates.
    *   **Index Calculation:** Compute the 1D array index from the 3D local coordinates. The most common layout is row-major (Z-major or Y-major depending on axis convention).
        *   **Y-major (common in JS/Unity):**
            $$ \text{index} = lx + ly \cdot S_x + lz \cdot S_x \cdot S_y $$
            [S3, unofficial]
        *   **Z-major (common in C++/Minecraft-style):**
            $$ \text{index} = ly \cdot S_x \cdot S_z + lz \cdot S_x + lx $$
            [S5, S7, unofficial]
    *   **Bounds Checking:** Verify that $0 \le lx < S_x$, $0 \le ly < S_y$, and $0 \le lz < S_z$. If out of bounds, return a default value (e.g., Air) or ignore the write [S3, S5, unofficial].

4.  **Update Notification:** If the voxel data is modified, mark the chunk as "dirty" or "modified" to trigger a mesh regeneration or neighbor update [S4, S6, unofficial].

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
        self.is_dirty = True

    def get_index(self, lx, ly, lz):
        # Y-major layout: index = x + y * sx + z * sx * sy
        return lx + ly * self.size_x + lz * self.size_x * self.size_y

    def set_voxel(self, lx, ly, lz, value):
        if not (0 <= lx < self.size_x and 0 <= ly < self.size_y and 0 <= lz < self.size_z):
            return
        idx = self.get_index(lx, ly, lz)
        self.data[idx] = value
        self.is_dirty = True

    def get_voxel(self, lx, ly, lz):
        if not (0 <= lx < self.size_x and 0 <= ly < self.size_y and 0 <= lz < self.size_z):
            return 0 # Air
        idx = self.get_index(lx, ly, lz)
        return self.data[idx]

class VoxelWorld:
    def __init__(self, chunk_size_x=16, chunk_size_y=256, chunk_size_z=16):
        self.chunk_size_x = chunk_size_x
        self.chunk_size_y = chunk_size_y
        self.chunk_size_z = chunk_size_z
        self.chunks = {} # Map from (cx, cy, cz) to Chunk

    def _get_chunk_key(self, cx, cy, cz):
        return (cx, cy, cz)

    def _get_chunk(self, cx, cy, cz):
        key = self._get_chunk_key(cx, cy, cz)
        if key not in self.chunks:
            self.chunks[key] = Chunk(cx, cy, cz, self.chunk_size_x, self.chunk_size_y, self.chunk_size_z)
        return self.chunks[key]

    def set_voxel(self, x, y, z, value):
        # Calculate chunk indices
        cx = x // self.chunk_size_x
        cy = y // self.chunk_size_y
        cz = z // self.chunk_size_z
        
        # Calculate local coordinates
        lx = x % self.chunk_size_x
        ly = y % self.chunk_size_y
        lz = z % self.chunk_size_z
        
        chunk = self._get_chunk(cx, cy, cz)
        chunk.set_voxel(lx, ly, lz, value)

    def get_voxel(self, x, y, z):
        cx = x // self.chunk_size_x
        cy = y // self.chunk_size_y
        cz = z // self.chunk_size_z
        
        lx = x % self.chunk_size_x
        ly = y % self.chunk_size_y
        lz = z % self.chunk_size_z
        
        key = self._get_chunk_key(cx, cy, cz)
        if key not in self.chunks:
            return 0 # Air
        
        chunk = self.chunks[key]
        return chunk.get_voxel(lx, ly, lz)
```

**Parameters:**
*   **Chunk Size:** Typically $16^3$, $32^3$, or non-cubic dimensions like $16 \times 256 \times 16$ (Minecraft-style) [S5, S6, unofficial].
    *   $16^3$ or $32^3$ are common for uniform voxel engines [S6, unofficial].
    *   $16 \times 256 \times 16$ is used to keep vertical data small for fast re-meshing while maintaining horizontal resolution [S5, unofficial].
*   **Chunk Shift:** If chunk size is a power of two, the bit shift amount is $\log_2(\text{ChunkSize})$ (e.g., 5 for 32) [S2, unofficial].
*   **Max Chunk Amount:** In flat-array implementations, the maximum number of chunks along each axis determines the stride for global indexing [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Duplicate Block Detection:** If storing individual block objects rather than a dense grid, reference equality checks may fail for logically identical blocks created separately [S1, unofficial].
*   **Memory Management:** In languages like C++, copying chunk instances by value (e.g., in iteration loops) can cause memory disasters due to the large size of the block ID array and associated OpenGL vertex data. Smart pointers or references should be used [S7, unofficial].
*   **Boundary Handling:** Accessing voxels at chunk boundaries requires neighbor chunk lookups. Implementations must handle cases where a neighbor chunk is not yet loaded or is out of bounds (returning Air or a boundary policy value) [S3, unofficial].
*   **Chunk Size Trade-offs:** Too large a chunk size leads to slow remeshing times when changes occur; too small a size results in excessive triangle counts and overhead [S6, unofficial].

## Subtopics

### [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md)
A Sparse Voxel Octree (SVO) is a hierarchical data structure that represents the geometry of a 3D voxelized scene by recursively decomposing space only where active voxels exist, effectively eliminating empty regions. It solves the memory inefficiency of dense grids by storing only the necessary nodes. Unlike flat grid structures, it provides tight compression of sparse spaces. It differs from its sibling [Sparse Voxel Octree with Bricks](2354-sparse-voxel-octree-with-bricks.md) by focusing on a compact, pointer-based node structure optimized for direct shader traversal rather than grouped bricks for cache locality.

### [Terrain Chunk Data Structure](2420-terrain-chunk-data-structure.md)
The Terrain Chunk Data Structure represents terrain as a 4-8 hierarchy of triangular patches, where each patch contains hundreds of triangles. This grouping enables efficient batched rendering, reduces mesh adaptation cost, supports view frustum culling, serves as a paging unit, and imposes spatial locality for compression. Unlike the parent [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md) which manages discrete voxel IDs, this method manages continuous geometric surfaces grouped into chunks. It is distinct from [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by focusing on surface meshes rather than volumetric occupancy, and from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by using a bintree hierarchy of patches instead of fixed 3D grids.

### [Chunk-Based World Storage](2858-chunk-based-world-storage.md)
Chunk-Based World Storage maps global voxel coordinates to specific chunks and local voxel indices to enable efficient storage and retrieval in a spatially partitioned voxel world. It solves the problem of managing infinite or large-scale voxel data by dividing the world into fixed-size 3D blocks, optimizing memory locality and access patterns. Unlike [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) which uses hierarchical pointers, this method relies on fixed-size dense grids within chunks, and differs from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by explicitly managing the coordinate translation logic for chunk boundaries.

### [Chunk Border Neighbor Access](2860-chunk-border-neighbor-access.md)
Chunk Border Neighbor Access is a strategy for handling neighbor lookups at chunk boundaries in chunk-based voxel data structures by providing a read-only neighborhood view trait. It solves the problem of distinguishing between internal voxels and boundary voxels during operations like meshing or rendering, treating chunk borders as semantically identical to internal voxels. This approach separates the logic of neighbor retrieval from the logic of neighbor occlusion or processing, allowing the meshing or rendering code to query local and cross-chunk neighbors through a uniform interface.

### [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md)
Compact Voxel Section Storage is a memory-optimized representation for a fixed-size 3D block of voxels (a "section") that dynamically selects between three storage modes—Empty, Uniform, and Palette—to minimize heap usage based on content entropy. It sets itself apart from sibling methods like [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by maintaining a dense grid layout for locality while using palette indexing to compress mixed-material sections, and differs from [Chunk-based World Storage](2858-chunk-based-world-storage.md) by operating at the finer granularity of a 16³ section rather than a larger world chunk.

### [Chunk Data Modification & Queue Management](4427-chunk-data-modification-queue-management.md)
This method handles the mutation of voxel data within a chunk-based voxel world and manages the subsequent regeneration of geometry for affected areas. It solves the problem of maintaining consistency between raw voxel data and the rendered mesh when user edits or game logic modify the world state. It differs from [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by operating on fixed-size dense grids rather than hierarchical trees, and from [Compact Voxel Section Storage](4219-compact-voxel-section-storage.md) by explicitly managing a modification queue for mesh updates rather than just storage compression.

### [Voxel Chunk Data Management and Residency](6751-voxel-chunk-data-management-and-residency.md)
This method manages the memory residency of voxel chunks in an infinite voxel world by tracking state transitions between raw, uniform, and compressed forms to reduce memory footprint for distant chunks. It solves the problem of unbounded memory usage in dense voxel grids by converting raw voxel arrays into compact representations (Uniform or Compressed) and dropping the resident raw array, while maintaining data consistency through versioning and mutation flags. Unlike sibling structures that focus on spatial partitioning or octree traversal, this method operates on the data lifecycle within a fixed-size chunk, using distance-keyed asynchronous compression to prioritize memory savings for far chunks.

### [Clustered Grid Cell (CGC) Data Structure](9217-clustered-grid-cell-cgc-data-structure.md)
The Clustered Grid Cell (CGC) data structure is a memory-efficient representation for isosurface rendering that stores active grid cells directly as rendering primitives, eliminating the need for the original volume scalar data during the rendering phase. It sets itself apart from previous cell rasterization approaches, such as Liu et al.'s method, by storing eight corner scalar values with each cell, allowing full rendering without GPU-resident volume textures, while achieving approximately 37% of the memory usage of typical efficient mesh-based representations.

### Voxel Layer System
A data structure system allowing non-destructive changes to voxel worlds by storing voxels in separate VoxelLayer structures rather than directly in the chunk array, enabling toggling visibility of different content sets (e.g., terrain vs. castle) [S8, unofficial].
*   **Definition:** Voxels are moved from the chunk array to separate `VoxelLayer` structures [S8, unofficial].
*   **Step:** Assign voxels to specific layers (e.g., background for terrain, top for castles) and toggle visibility of specific layers to show/hide content without modifying the underlying voxel data [S8, unofficial].
*   **Parameter:** Default Layer Count is two (background and top), though the system supports more [S8, unofficial].
*   **Layout:** Each chunk contains multiple layers (e.g., 'background' for terrain, 'top' for structures) [S8, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   **Access:** $O(1)$ for chunk lookup and local indexing, assuming a hash map or direct array access for chunks [S2, S5, unofficial].
    *   **Meshing:** $O(N)$ where $N$ is the number of voxels in the chunk, but typically much faster than global meshing because only modified chunks are regenerated [S6, unofficial].
*   **Memory Complexity:** $O(C \cdot S^3)$ where $C$ is the number of active chunks and $S$ is the chunk size. Memory usage scales linearly with the number of loaded chunks [S5, S6, unofficial].
*   **Measured Performance:**
    *   On an RTX 3050 laptop, rendering ~5,000 blocks across multiple chunks achieved over 200 FPS [S7, unofficial].
    *   Greedy meshing within chunks can reduce triangle count by 10× compared to individual block rendering [S5, unofficial].
*   **Scaling:** Scales well to infinite worlds by unloading distant chunks and loading new ones as the camera moves [S6, S7, unofficial].
*   **Strengths:**
    *   Efficient incremental updates (only modified chunks need remeshing) [S6, unofficial].
    *   Good spatial locality for cache performance [S2, unofficial].
    *   Simplifies world serialization (save/load chunks individually) [S6, unofficial].
*   **Limitations:**
    *   Fixed chunk size requires tuning; too large causes slow remeshing, too small increases overhead [S6, unofficial].
    *   Boundary handling between chunks adds complexity for operations like meshing or neighbor queries .
    *   Dense storage within chunks wastes memory if the world is very sparse (compared to SVO) [S1, S2, unofficial].
*   **When to use:** Ideal for voxel engines with frequent local modifications, large but not extremely sparse worlds, or when simple, predictable memory management is preferred over hierarchical compression.
*   **When not to use:** For extremely sparse worlds where memory efficiency is critical (use SVO), or for worlds requiring complex hierarchical LOD beyond chunk-level (use N³-tree or Bricked Volume Hierarchy).

## Background
The chunk-based approach is a foundational technique in voxel engine development, popularized by games like Minecraft. It addresses the limitations of storing entire worlds in a single data structure by partitioning space into manageable units. Early implementations in JavaScript and WebGL demonstrated the feasibility of this approach for web-based voxel engines [S1, S5, unofficial]. Subsequent developments in C++ and Unity environments refined the technique, adding optimizations like smart pointers for memory management [S7, unofficial], flat array indexing for performance [S2, unofficial], and integration with job systems for parallel meshing [S4, unofficial]. The method has become a standard pattern for managing voxel data in real-time applications, with variations focusing on layering [S8, unofficial], compact storage , and residency management .

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
[S3] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S4] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S5] Build a 3D Voxel Engine in WebGL | Tutorials, https://www.mysimulator.uk/content/tutorials/voxel-engine.html, fetched 2026-10-27 (unofficial)
[S6] kotychok/mutiny, https://github.com/kotychok/mutiny, fetched 2026-10-27 (unofficial)
[S7] Creating a Voxel Engine with OpenGL | Walker McGilvary, https://theslabby.github.io/2023/12/24/start-voxelengine.html, fetched 2026-10-28 (unofficial)
[S8] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
