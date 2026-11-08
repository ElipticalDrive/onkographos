# Coordinate Conversion Utilities

Also called: FVoxelCoordinates, LocalToVoxel, Voxel Chunk Coordinate Conversion, Voxel Coordinate Conversion, VoxelToLocal, voxelVector

## Overview
Coordinate Conversion Utilities provide the mathematical and algorithmic primitives to map positions between world space, chunk space, and local voxel space within a dense voxel grid. They solve the problem of addressing volumetric data stored in spatially partitioned chunks, ensuring that global coordinates are correctly resolved to local indices for memory access and that local indices are correctly mapped back to world coordinates for rendering or physics [S1, S2, unofficial]. This method is distinguished from its siblings by focusing on the fundamental integer arithmetic of chunk-based indexing and axis alignment, rather than spherical mapping [Siblings: Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md), incremental back-rotation [Siblings: Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md), or 2D texture packing [Siblings: PixelGrid2D Construction](9430-pixelgrid2d-construction.md).

## How it works
The method consists of two primary conversion paths: World-to-Voxel (global) and Local-to-Global (chunk-relative). It relies on the parameters defined in [Voxel Space Setup](3520-voxel-space-setup.md), specifically the `ChunkSize` (integer, e.g., 16 or 32) and `VoxelSize` (float, e.g., 1.0).

### Data Structures
*   **WorldPosition**: A 3D vector $(x, y, z)$ in world units.
*   **ChunkCoord**: A 3D integer vector $(cx, cy, cz)$ representing the chunk index in the global grid.
*   **LocalVoxel**: A 3D integer vector $(lx, ly, lz)$ representing the voxel index within a chunk, where $0 \le l_i < \text{ChunkSize}$.
*   **GlobalVoxel**: A 3D integer vector $(gx, gy, gz)$ representing the absolute voxel index in the entire world.

### Step 1: World to Global Voxel Coordinates
Converts a continuous world position to an integer global voxel index. This typically involves translating the world origin to the voxel grid origin and dividing by voxel size.

1.  Let $P_w = (x, y, z)$ be the world position.
2.  Let $O_w = (ox, oy, oz)$ be the world-space origin of the voxel grid (often the minimum corner of the bounding box).
3.  Let $S_v$ be the voxel size (world units per voxel).
4.  Calculate relative position: $P_{rel} = P_w - O_w$.
5.  Calculate global voxel coordinates:
    $$
    G = \left( \lfloor \frac{P_{rel.x}}{S_v} \rfloor, \lfloor \frac{P_{rel.y}}{S_v} \rfloor, \lfloor \frac{P_{rel.z}}{S_v} \rfloor \right)
    $$
    *Note: Floor division is used to ensure consistent indexing for negative coordinates relative to the grid origin.* [S2, unofficial]

### Step 2: Global Voxel to Chunk and Local Coordinates
Splits the global voxel index into the chunk it belongs to and the local offset within that chunk.

1.  Let $G = (gx, gy, gz)$ be the global voxel coordinate.
2.  Let $C_s$ be the `ChunkSize`.
3.  Calculate Chunk Coordinate:
    $$
    C = \left( \lfloor \frac{gx}{C_s} \rfloor, \lfloor \frac{gy}{C_s} \rfloor, \lfloor \frac{gz}{C_s} \rfloor \right)
    $$
4.  Calculate Local Voxel Coordinate:
    $$
    L = (gx \pmod{C_s}, gy \pmod{C_s}, gz \pmod{C_s})
    $$
5.  **Handle Negative Modulo**: In many programming languages, the modulo operator `%` returns a negative result for negative operands. To ensure $L$ is always within $[0, C_s - 1]$:
    ```python
    if L.x < 0: L.x += C_s
    if L.y < 0: L.y += C_s
    if L.z < 0: L.z += C_s
    ```
    [S2, unofficial]

### Step 3: Local Voxel to Global Voxel Coordinates
Reconstructs the global index from chunk and local coordinates.

1.  Let $C = (cx, cy, cz)$ be the chunk coordinate.
2.  Let $L = (lx, ly, lz)$ be the local voxel coordinate.
3.  Calculate Global Voxel Coordinate:
    $$
    G = (cx \cdot C_s + lx, cy \cdot C_s + ly, cz \cdot C_s + lz)
    $$
    [S2, unofficial]

### Step 4: Local Voxel to World Coordinates
Converts a local voxel index (and its chunk) back to a world position, typically at the voxel center.

1.  Let $C = (cx, cy, cz)$ be the chunk coordinate.
2.  Let $L = (lx, ly, lz)$ be the local voxel coordinate.
3.  Calculate Global Voxel Coordinate $G$ using Step 3.
4.  Calculate World Position (Voxel Center):
    $$
    P_w = O_w + (G + 0.5) \cdot S_v
    $$
    *Note: Adding 0.5 shifts the index from the voxel's minimum corner to its center.* [S2, unofficial]

### Pseudocode
The following Python-like pseudocode implements the core utilities described in [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def world_to_chunk(world_pos, voxel_origin, voxel_size, chunk_size):
    """
    Converts a world position to a chunk coordinate.
    """
    # Step 1: World to Global Voxel
    rel_x = world_pos.x - voxel_origin.x
    rel_y = world_pos.y - voxel_origin.y
    rel_z = world_pos.z - voxel_origin.z
    
    # Floor division for global voxel indices
    global_voxel_x = int(rel_x / voxel_size)
    global_voxel_y = int(rel_y / voxel_size)
    global_voxel_z = int(rel_z / voxel_size)
    
    # Step 2: Global Voxel to Chunk
    chunk_x = int(global_voxel_x / chunk_size)
    chunk_y = int(global_voxel_y / chunk_size)
    chunk_z = int(global_voxel_z / chunk_size)
    
    return (chunk_x, chunk_y, chunk_z)

def world_to_local_voxel(world_pos, voxel_origin, voxel_size, chunk_size):
    """
    Converts a world position to local voxel coordinates within a chunk.
    Returns (chunk_coord, local_voxel_coord).
    """
    # Step 1: World to Global Voxel
    rel_x = world_pos.x - voxel_origin.x
    rel_y = world_pos.y - voxel_origin.y
    rel_z = world_pos.z - voxel_origin.z
    
    global_voxel_x = int(rel_x / voxel_size)
    global_voxel_y = int(rel_y / voxel_size)
    global_voxel_z = int(rel_z / voxel_size)
    
    # Step 2: Global Voxel to Chunk
    chunk_x = int(global_voxel_x / chunk_size)
    chunk_y = int(global_voxel_y / chunk_size)
    chunk_z = int(global_voxel_z / chunk_size)
    
    # Step 3: Global Voxel to Local Voxel
    local_x = global_voxel_x % chunk_size
    local_y = global_voxel_y % chunk_size
    local_z = global_voxel_z % chunk_size
    
    # Handle negative modulo results
    if local_x < 0: local_x += chunk_size
    if local_y < 0: local_y += chunk_size
    if local_z < 0: local_z += chunk_size
    
    return ((chunk_x, chunk_y, chunk_z), (local_x, local_y, local_z))

def local_voxel_to_world(chunk_coord, local_voxel, voxel_origin, voxel_size, chunk_size):
    """
    Converts a local voxel coordinate (with its chunk) back to world space.
    Returns the world position at the center of the voxel.
    """
    cx, cy, cz = chunk_coord
    lx, ly, lz = local_voxel
    
    # Step 3: Local to Global
    global_voxel_x = cx * chunk_size + lx
    global_voxel_y = cy * chunk_size + ly
    global_voxel_z = cz * chunk_size + lz
    
    # Step 4: Global to World (Center)
    world_x = voxel_origin.x + (global_voxel_x + 0.5) * voxel_size
    world_y = voxel_origin.y + (global_voxel_y + 0.5) * voxel_size
    world_z = voxel_origin.z + (global_voxel_z + 0.5) * voxel_size
    
    return (world_x, world_y, world_z)
```

### Edge Cases and Pitfalls
*   **Negative Coordinates**: The modulo operation must be handled carefully. Standard `%` in C++/Python behaves differently for negative numbers. The explicit check `if (local < 0) local += chunk_size` is required for correctness in languages where `%` preserves the sign of the dividend [S2, unofficial].
*   **Precision**: When converting from world to voxel, floating-point precision errors near chunk boundaries can cause jitter. It is recommended to use `floor` or integer division consistently rather than rounding [S2, unofficial].
*   **Chunk Boundaries**: Accessing a voxel at the edge of a chunk (e.g., index `ChunkSize - 1`) must correctly map to the global index without overflow. The formula $G = C \cdot C_s + L$ ensures this if $C_s$ is constant.

## Subtopics

### [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md)
Sliced Volume Coordinate Mapping converts 3D voxel coordinates into 2D texture coordinates for volumetric data stored as a tiled grid of 2D slices within a single image. It solves the problem of accessing 3D volumetric data in renderers that only support 2D texture sampling by mapping a 3D point to two adjacent 2D slice coordinates. It is distinguished from nearest-neighbor sampling by using linear interpolation between slices to reduce banding artifacts.

### Adaptive Coordinate Conversion
Adaptive Coordinate Conversion is a technique used by Lim et al. to visualize 3D ultrasound datasets, where points within the native pyramidal ultrasound coordinate system were adaptively converted on the fly to a Cartesian grid [S3]. This allows for efficient raycasting on hardware that expects regular grid structures, adapting the coordinate transformation dynamically based on the viewing angle or dataset density.

### Coordinate System Bit Encoding
Coordinate System Bit Encoding is a method for representing and comparing 3D coordinate systems using a compact 32-bit integer enum, facilitating axis alignment checks and rotation requirements between different coordinate spaces [S4, unofficial].

*   **Layout**: The `CoordinateSystem` is a `uint32_t` enum with 16 valid values (1-16) representing axis directions. Value 0 is `UNSPECIFIED`.
    *   Bit 0: Left (0) / Right (1)
    *   Bit 1: Down (0) / Up (1)
    *   Bit 2: Back (0) / Front (1)
    *   Examples: LDB=1, RDB=2, LUB=3, RUB=4, LDF=5, RDF=6, LUF=7, RUF=8, LFD=9, RFD=10, LFU=11, RFU=12, LBD=13, RBD=14, LBU=15, RBU=16.
*   **Logic**:
    *   `axesMatch(a, b)`: Determines if axes align by converting enums to 0-based indices ($aNum = a-1, bNum = b-1$). If either is `UNSPECIFIED` (<0), returns `{true, true, true}`. Otherwise, compares bit $i$ of $aNum$ and $bNum$ for $i=0,1,2$. Returns a 3-element bool array indicating match per axis.
    *   `needRotation(a, b)`: Checks if a $\pm \pi/2$ rotation is required by comparing bit 3 of the 0-based indices. If either is `UNSPECIFIED`, returns `false`. Otherwise, returns `true` if bit 3 differs.

## Operating characteristics
*   **Time Complexity**: $O(1)$ for all conversion operations. The calculations involve basic arithmetic (addition, multiplication, division, modulo) and bitwise operations.
*   **Memory Complexity**: $O(1)$ auxiliary space. No large data structures are required for the conversion itself, only the parameters ($ChunkSize$, $VoxelSize$, $Origin$).
*   **Performance**: Extremely fast, suitable for per-voxel operations in shaders or high-frequency CPU updates. The bitwise encoding for coordinate systems [S4, unofficial] is particularly efficient for runtime checks.
*   **Strengths**: Simple to implement, robust handling of negative coordinates, and efficient chunk-based memory access patterns.
*   **Limitations**: Does not handle non-Cartesian grids (e.g., spherical) without modification. Precision issues may arise in very large worlds if floating-point coordinates are used without careful origin management.

## Background
Coordinate conversion utilities are fundamental to voxel-based rendering and simulation. Early implementations in voxel engines like `voxel-engine-stackgl` [S1, unofficial] established the basic world-to-chunk mapping. More complex systems, such as those in Unreal Engine-based voxel worlds [S2, unofficial], introduced robust handling of chunk boundaries and negative coordinates. In medical visualization, adaptive coordinate conversion was developed to handle non-Cartesian data sources like ultrasound [S3]. The bit-encoding scheme for coordinate systems [S4, unofficial] is a more recent development aimed at standardizing axis alignment in file formats and engine internals.

## Sources
[S1] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S2] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S3] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S4] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
