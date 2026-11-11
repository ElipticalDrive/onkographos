# Coordinate Conversion Utilities

Also called: FVoxelCoordinates, LocalToVoxel, Voxel Chunk Coordinate Conversion, Voxel Coordinate Conversion, VoxelToLocal, voxelVector

## Overview
Coordinate Conversion Utilities provide the mathematical and algorithmic primitives for mapping positions between world space, chunk space, and local voxel space within a hierarchical voxel grid system. They solve the problem of efficiently addressing volumetric data stored in discrete chunks by handling the translation of continuous world coordinates into discrete integer indices, while managing boundary conditions such as negative coordinates and chunk alignment. What sets these utilities apart from [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md) is their reliance on Cartesian grid topology rather than spherical harmonics or polar coordinates, and from [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md) by operating on static axis-aligned bounding boxes rather than dynamic camera-relative rotations.

## How it works
The core functionality involves bidirectional conversion between world-space positions and voxel indices, often mediated by a chunking layer to support sparse or large-scale worlds. The implementation relies on floor division for coordinate-to-index mapping and modulo arithmetic for localizing coordinates within chunks.

### Inputs and Outputs
*   **Inputs:**
    *   `WorldPos`: A 3D vector $(x, y, z)$ in continuous world units.
    *   `VoxelSize`: The physical size of a single voxel (scalar or vector).
    *   `ChunkSize`: The number of voxels along one axis in a chunk (scalar).
    *   `ChunkCoord`: The integer coordinates of the chunk containing the point.
    *   `LocalPos`: A 3D vector of integer voxel indices relative to the chunk origin.
*   **Outputs:**
    *   `ChunkCoord`: Integer 3D vector identifying the chunk.
    *   `GlobalVoxelCoord`: Integer 3D vector identifying the voxel in the global grid.
    *   `LocalVoxelCoord`: Integer 3D vector identifying the voxel within the chunk.
    *   `WorldPos`: Continuous 3D vector reconstructed from voxel indices.

### Data Structures
*   **FIntVector / VoxelCoord:** A 3-element integer vector $(X, Y, Z)$.
*   **FVector / WorldPos:** A 3-element floating-point vector $(x, y, z)$.
*   **CoordinateSystem Enum:** A 32-bit unsigned integer (`uint32_t`) encoding axis directions and orientations [S4, unofficial].

### Numbered Steps

#### 1. World to Global Voxel Coordinates
Convert a continuous world position to a global integer voxel index.
1.  Normalize the world position by the voxel size: $V = \frac{WorldPos}{VoxelSize}$.
2.  Apply floor division to obtain integer indices:
    $$GlobalVoxelCoord.X = \lfloor V.X \rfloor$$
    $$GlobalVoxelCoord.Y = \lfloor V.Y \rfloor$$
    $$GlobalVoxelCoord.Z = \lfloor V.Z \rfloor$$
    [S2, unofficial]

#### 2. Global Voxel to Chunk and Local Coordinates
Decompose the global voxel index into chunk and local components.
1.  Calculate Chunk Coordinates using floor division:
    $$ChunkCoord.X = \lfloor \frac{GlobalVoxelCoord.X}{ChunkSize} \rfloor$$
    $$ChunkCoord.Y = \lfloor \frac{GlobalVoxelCoord.Y}{ChunkSize} \rfloor$$
    $$ChunkCoord.Z = \lfloor \frac{GlobalVoxelCoord.Z}{ChunkSize} \rfloor$$
    [S2, unofficial]
2.  Calculate Local Voxel Coordinates using modulo arithmetic. To handle negative coordinates correctly (ensuring the result is always positive within the chunk range $[0, ChunkSize-1]$):
    $$LocalPos.X = GlobalVoxelCoord.X \pmod{ChunkSize}$$
    $$LocalPos.Y = GlobalVoxelCoord.Y \pmod{ChunkSize}$$
    $$LocalPos.Z = GlobalVoxelCoord.Z \pmod{ChunkSize}$$
    If $LocalPos.X < 0$, then $LocalPos.X \leftarrow LocalPos.X + ChunkSize$.
    If $LocalPos.Y < 0$, then $LocalPos.Y \leftarrow LocalPos.Y + ChunkSize$.
    If $LocalPos.Z < 0$, then $LocalPos.Z \leftarrow LocalPos.Z + ChunkSize$.
    [S2, unofficial]

#### 3. Local Voxel to Global Voxel Coordinates
Reconstruct the global index from chunk and local components.
1.  Calculate the global index:
    $$GlobalVoxelCoord.X = ChunkCoord.X \times ChunkSize + LocalPos.X$$
    $$GlobalVoxelCoord.Y = ChunkCoord.Y \times ChunkSize + LocalPos.Y$$
    $$GlobalVoxelCoord.Z = ChunkCoord.Z \times ChunkSize + LocalPos.Z$$
    [S2, unofficial]

#### 4. Local Voxel to World Position
Convert a local voxel index (and its chunk) back to a continuous world position, typically representing the voxel center.
1.  Calculate the chunk's world-space origin (minimum corner):
    $$ChunkOrigin.X = ChunkCoord.X \times ChunkSize \times VoxelSize$$
    $$ChunkOrigin.Y = ChunkCoord.Y \times ChunkSize \times VoxelSize$$
    $$ChunkOrigin.Z = ChunkCoord.Z \times ChunkSize \times VoxelSize$$
    [S2, unofficial]
2.  Calculate the voxel's offset within the chunk:
    $$VoxelOffset.X = LocalPos.X \times VoxelSize$$
    $$VoxelOffset.Y = LocalPos.Y \times VoxelSize$$
    $$VoxelOffset.Z = LocalPos.Z \times VoxelSize$$
    [S2, unofficial]
3.  Add half a voxel size to center the coordinate:
    $$WorldPos.X = ChunkOrigin.X + VoxelOffset.X + \frac{VoxelSize}{2}$$
    $$WorldPos.Y = ChunkOrigin.Y + VoxelOffset.Y + \frac{VoxelSize}{2}$$
    $$WorldPos.Z = ChunkOrigin.Z + VoxelOffset.Z + \frac{VoxelSize}{2}$$
    [S2, unofficial]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def world_to_global_voxel(world_pos: Vector3, voxel_size: float) -> IntVector3:
    """
    Converts a continuous world position to global integer voxel coordinates.
    """
    # Normalize by voxel size
    normalized_x = world_pos.x / voxel_size
    normalized_y = world_pos.y / voxel_size
    normalized_z = world_pos.z / voxel_size
    
    # Floor division to get integer indices
    return IntVector3(
        floor(normalized_x),
        floor(normalized_y),
        floor(normalized_z)
    )

def global_voxel_to_chunk_and_local(global_voxel: IntVector3, chunk_size: int) -> Tuple[IntVector3, IntVector3]:
    """
    Decomposes a global voxel index into chunk coordinates and local voxel coordinates.
    Handles negative coordinates by adjusting the modulo result.
    """
    # Calculate chunk coordinates
    chunk_x = floor(global_voxel.x / chunk_size)
    chunk_y = floor(global_voxel.y / chunk_size)
    chunk_z = floor(global_voxel.z / chunk_size)
    
    # Calculate local coordinates using modulo
    local_x = global_voxel.x % chunk_size
    local_y = global_voxel.y % chunk_size
    local_z = global_voxel.z % chunk_size
    
    # Handle negative modulo results (e.g., -1 % 8 = -1 in Python/C++, but we want 7)
    if local_x < 0:
        local_x += chunk_size
    if local_y < 0:
        local_y += chunk_size
    if local_z < 0:
        local_z += chunk_size
        
    return (IntVector3(chunk_x, chunk_y, chunk_z), IntVector3(local_x, local_y, local_z))

def local_voxel_to_global(chunk_coord: IntVector3, local_pos: IntVector3, chunk_size: int) -> IntVector3:
    """
    Reconstructs global voxel coordinates from chunk and local indices.
    """
    return IntVector3(
        chunk_coord.x * chunk_size + local_pos.x,
        chunk_coord.y * chunk_size + local_pos.y,
        chunk_coord.z * chunk_size + local_pos.z
    )

def local_voxel_to_world(chunk_coord: IntVector3, local_pos: IntVector3, chunk_size: int, voxel_size: float) -> Vector3:
    """
    Converts local voxel coordinates to world-space position (voxel center).
    """
    # Calculate chunk origin (minimum corner)
    chunk_origin_x = chunk_coord.x * chunk_size * voxel_size
    chunk_origin_y = chunk_coord.y * chunk_size * voxel_size
    chunk_origin_z = chunk_coord.z * chunk_size * voxel_size
    
    # Calculate voxel offset
    voxel_offset_x = local_pos.x * voxel_size
    voxel_offset_y = local_pos.y * voxel_size
    voxel_offset_z = local_pos.z * voxel_size
    
    # Add half voxel size to center
    return Vector3(
        chunk_origin_x + voxel_offset_x + voxel_size / 2.0,
        chunk_origin_y + voxel_offset_y + voxel_size / 2.0,
        chunk_origin_z + voxel_offset_z + voxel_size / 2.0
    )
```

### Parameters and Typical Values
*   **VoxelSize:** Typically $1.0$ unit in world space, but varies by scene scale.
*   **ChunkSize:** Commonly powers of 2 (e.g., 16, 32, 64) to optimize bitwise operations for division and modulo, though not strictly required by the formulas above.
*   **CoordinateSystem Enum:** 16 valid values (1-16) representing axis directions. Bit 0: Left(0)/Right(1). Bit 1: Down(0)/Up(1). Bit 2: Back(0)/Front(1). Value 0 is UNSPECIFIED. Examples: LDB=1, RDB=2, LUB=3, RUB=4, LDF=5, RDF=6, LUF=7, RUF=8, LFD=9, RFD=10, LFU=11, RFU=12, LBD=13, RBD=14, LBU=15, RBU=16 [S4, unofficial].

### Edge Cases and Pitfalls
*   **Negative Coordinates:** Standard modulo operators in many languages (C++, Python) return negative results for negative operands. The algorithm must explicitly add `ChunkSize` to negative local coordinates to ensure they fall within $[0, ChunkSize-1]$ [S2, unofficial].
*   **Precision Loss:** Converting from world space to voxel space and back introduces quantization error. The reconstructed world position will be the center of the voxel, not the original input point.
*   **Boundary Conditions:** Points exactly on chunk boundaries or voxel edges must be handled consistently with the floor function to ensure adjacent voxels/chunks do not overlap or leave gaps.

## Subtopics

### [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md)
Sliced Volume Coordinate Mapping converts 3D voxel coordinates into 2D texture coordinates for volumetric data stored as a tiled grid of 2D slices within a single 2D image. This technique enables GPU texture sampling of 3D data using standard 2D texture units, which is necessary when hardware 3D texture support is unavailable or inefficient. It differs from standard 3D texture sampling by flattening the volume into a 2D grid of slices, requiring specific coordinate mapping logic to resolve 3D coordinates to 2D image pixels.

### [Voxel Index Computation](11238-voxel-index-computation.md)
Computes discrete voxel indices for a batch of 3D points by mapping their continuous coordinates into a structured grid defined by bounding boxes and voxel dimensions. It differs from [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md) by operating on a unified 3D grid structure rather than slicing the volume into 2D planes, making it suitable for dense volumetric queries and ray marching initialization.

### [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md)
Metric Coordinate Conversion decomposes a continuous world-space position into a discrete integer component (whole meters) and a continuous fractional component (sub-meter precision). This decomposition enables efficient spatial indexing and caching in voxel-based rendering systems by separating coarse grid alignment from fine-grained sampling offsets. It differs from [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md) by operating on absolute metric units rather than slice indices, and from [Voxel Index Computation](11238-voxel-index-computation.md) by preserving sub-voxel precision rather than truncating to integer voxel coordinates.

### Adaptive Coordinate Conversion
A technique used by Lim et al. to visualize 3D ultrasound datasets, where points within the native pyramidal ultrasound coordinate system were adaptively converted on the fly to a Cartesian grid [S3]. This allows for real-time visualization of non-Cartesian data sources without pre-processing the entire dataset into a regular grid.

### Coordinate System Bit Encoding
A method for encoding axis directions and orientations into a single 32-bit integer to facilitate efficient coordinate system transformations. The `CoordinateSystem` enum uses bits 0-2 to define the primary axis directions (Left/Right, Down/Up, Back/Front) and bit 3 to indicate if a $\pm \pi/2$ rotation is required. The `axesMatch` function determines if two coordinate systems align by comparing the lower 3 bits of their 0-based indices, while `needRotation` checks bit 3 to determine if a rotation is necessary [S4, unofficial].

### Coordinate System Conversion
Conversion of normalized Cartesian data $(x, y, z, r, g, b)$ to system coordinates $(h, w, \theta, r, g, b)$ for swept-volume display. The implementation requires two steps: 1) conversion of data from Descartes coordinates to cylindrical coordinates $(\theta, \rho, z, r, g, b)$; 2) conversion of data from cylindrical coordinates to system coordinates $(h, w, \theta, r, g, b)$ [S5]. This is used in specialized volumetric displays where the data needs to be mapped to a specific scanning geometry.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for all conversion operations. The calculations involve basic arithmetic operations (addition, multiplication, division, modulo) which are constant time relative to the number of points.
*   **Memory Complexity:** $O(1)$ auxiliary space. No additional data structures are required beyond the input and output vectors.
*   **Performance:** Extremely fast, suitable for per-voxel operations in ray marching or per-pixel operations in fragment shaders. The use of integer arithmetic for indices minimizes precision issues and leverages CPU/GPU integer units.
*   **Strengths:** Simple to implement, highly efficient, supports sparse data structures via chunking, handles negative coordinates correctly.
*   **Limitations:** Assumes axis-aligned bounding boxes. Does not handle rotated or non-Cartesian coordinate systems without additional transformation matrices (see [Incremental Back-Rotation for Voxel Mapping](6759-incremental-back-rotation-for-voxel-mapping.md)). Precision is limited by the voxel size; sub-voxel details are lost during the integer conversion step unless fractional components are preserved separately (see [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md)).

## Background
Coordinate conversion utilities are fundamental to voxel-based rendering and spatial data structures. The specific chunk-based approach described in [S2, unofficial] is common in procedural generation and large-scale voxel worlds (e.g., Minecraft-like engines) to manage memory and loading. The adaptive conversion for ultrasound data described in [S3] highlights the application of these utilities in medical imaging, where non-Cartesian data sources must be mapped to standard rendering pipelines. The bit-encoding scheme in [S4, unofficial] is used in file formats and coordinate system definitions to compactly store orientation information. The swept-volume conversion in [S5] represents a specialized application in volumetric display technology.

## Sources
[S1] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S2] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S3] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S4] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
[S5] PDF An Improved Design of 3D Swept-Volume Volumetric Display, https://jcomputers.us/vol9/jcp0901-34.pdf, fetched 2026-10-28
