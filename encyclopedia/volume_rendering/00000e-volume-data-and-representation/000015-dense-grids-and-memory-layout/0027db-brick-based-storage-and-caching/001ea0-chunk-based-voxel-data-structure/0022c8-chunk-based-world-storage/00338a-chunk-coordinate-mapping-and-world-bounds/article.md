# Chunk Coordinate Mapping and World Bounds

Also called: Chunk-based Voxelization and Position Mapping, Chunk-to-World Transform

## Overview
This method decomposes a global voxel position into map-space chunk coordinates and chunk-local voxel coordinates, enabling efficient memory management by dividing the voxel world into fixed-size chunks [S1, unofficial]. It converts between chunk grid indices and world coordinates and calculates the total world width covered by the chunk system [S2, unofficial]. Unlike [Chunk Coordinate Normalization](12959-chunk-coordinate-normalization.md), which actively corrects non-canonical inputs to prevent overflow, this method focuses on the primary translation between valid world, chunk, and local representations [S1, unofficial].

## How it works
**Inputs:**
- `pos`: A 3D integer vector representing the global voxel position in world space (e.g., `DNivec3`) [S1, unofficial].
- `chunkSize`: The number of voxels along one axis per chunk (e.g., 16, 32) [S1, S2, S3, unofficial].
- `chunkDistance`: The radius of loaded chunks around the viewer/player in chunk indices [S2, unofficial].

**Outputs:**
- `mapPos`: A 3D integer vector representing the chunk position in the global map grid [S1, unofficial].
- `chunkPos`: A 3D integer vector representing the local position within the identified chunk [S1, unofficial].
- `worldWidth`: The total spatial extent covered by the chunk system [S2, unofficial].

**Data Structures:**
- `pos`, `mapPos`, `chunkPos`: 3-component integer vectors.
- `chunkSize`: Integer.
- `chunkDistance`: Integer.

**Steps:**

1. **Calculate World Bounds:** Compute the total world width covered by the chunk system using the formula `worldWidth = chunkSize * 2 * chunkDistance` [S2, unofficial].
2. **Compute Chunk Index:** Determine which chunk contains the global voxel position by dividing the position by the chunk size and flooring the result: `mapPos_i = floor(pos_i / chunkSize)` for each axis $i$ [S1, unofficial]. Standard integer division truncates toward zero in many languages, so explicit floor division is required for negative coordinates to ensure correct chunk indexing [S3, unofficial].
3. **Compute Local Position:** Determine the voxel's offset within the identified chunk by subtracting the chunk's base world position from the global position: `chunkPos_i = pos_i - mapPos_i * chunkSize` [S1, unofficial]. In Python, this is equivalent to `pos_i % chunkSize` because the modulo operator uses floored division [G].
4. **Handle Negative Coordinate Mirroring (Implementation-Specific):** Some implementations invert the local position for negative coordinates to maintain consistent chunk indexing and avoid boundary errors [S3, unofficial]. This involves calculating a fractional remainder from the absolute coordinate (`frac_pos = abs(coord)*100 - floor(abs(coord))*100`) and applying `local_pos = chunkSize - 1 - frac_pos` when the coordinate is negative [S3, unofficial].
5. **Convert to World Coordinates:** Convert chunk indices back to world coordinates by multiplying the chunk position by the chunk size: `worldCoord_i = mapPos_i * chunkSize` [S2, unofficial]. For precise floating-point world positions including local offsets, combine the chunk base and local position: `final_coord_i = chunk_pos_i + local_pos_i / 100.0` for positive coordinates, or `final_coord_i = chunk_pos_i - local_pos_i / 100.0` for negative coordinates, where the divisor `100.0` reflects a specific implementation's internal scaling [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunk_coordinate_mapping(pos, chunk_size):
    """
    Decomposes a global voxel position into chunk coordinates and local voxel coordinates.
    
    Args:
        pos: Global voxel position as a list of 3 integers.
        chunk_size: Number of voxels per chunk axis (int).
        
    Returns:
        map_pos: Chunk index in the global grid (list of 3 ints).
        local_pos: Local voxel position within the chunk (list of 3 ints).
    """
    map_pos = []
    local_pos = []
    for coord in pos:
        # Step 2: Compute Chunk Index
        # Python's // operator performs floor division, handling negatives correctly.
        # Other languages may require explicit floor adjustment.
        map_coord = coord // chunk_size
        map_pos.append(map_coord)
        # Step 3: Compute Local Position
        local_coord = coord - map_coord * chunk_size
        local_pos.append(local_coord)
    return map_pos, local_pos

def world_to_chunk(world_pos, chunk_size):
    """
    Converts world-space floating point coordinates to chunk indices and local offsets.
    
    Args:
        world_pos: World space position as a list of 3 floats.
        chunk_size: Size of a chunk in world units (float).
        
    Returns:
        chunk_idx: Chunk grid index (list of 3 ints).
        local_offset: Local position within the chunk (list of 3 floats).
    """
    chunk_idx = []
    local_offset = []
    for coord in world_pos:
        # Step 2: Compute chunk index
        chunk_c = int(coord // chunk_size) if coord >= 0 else -((-coord + chunk_size - 1) // chunk_size)
        chunk_idx.append(chunk_c)
        # Step 5: Compute local offset
        local_c = coord - chunk_c * chunk_size
        local_offset.append(local_c)
    return chunk_idx, local_offset
```

**Parameters:**
- `chunkSize`: Typical values are 16 or 32 voxels, balancing memory overhead per chunk against the granularity of loading/unloading [S1, S2, unofficial].
- `chunkDistance`: Typically 2 or more chunks, defining the radius of the active area around the viewer [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Negative Coordinates:** Standard integer division and modulo operations in many programming languages truncate toward zero, which can lead to incorrect chunk indices for negative world coordinates (e.g., `-1 // 16` might be `0` instead of `-1`) [S3, unofficial]. Explicit floor division or offset adjustments are required.
- **Boundary Errors:** Failing to handle the inversion or offset for negative coordinates can cause discontinuities in data access or rendering artifacts at chunk boundaries [S3, unofficial].
- **Precision:** When converting between integer voxel coordinates and floating-point world coordinates, precision loss can occur if not handled carefully, especially with large world coordinates [S3, unofficial].

## Subtopics

### [Chunk Coordinate Normalization](12959-chunk-coordinate-normalization.md)
Chunk Coordinate Normalization ensures that a world-space position is represented by a canonical pair of chunk indices and local coordinates, preventing coordinate ambiguity and overflow. It differs from Chunk Coordinate Mapping and World Bounds by actively correcting non-canonical inputs (where local coordinates fall outside the chunk bounds) rather than merely converting between existing valid representations. It serves as a prerequisite for density field sampling, ensuring that subsequent lookups operate on consistent spatial data.

### Chunk Distance Calculation
Calculates the squared distance between the viewer position and the center of a chunk in world units [S4, unofficial]. This is used to determine if a chunk should be loaded or unloaded based on the viewer's proximity [S4, unofficial]. The calculation involves determining the chunk's center in world space via `ChunkToWorldCenter(ChunkCoord, ChunkSize, VoxelSize)` and then computing the Euclidean distance or squared distance to the viewer's position [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Coordinate mapping is $O(1)$ per axis, involving basic arithmetic operations (division, modulo) [S1, S2, unofficial]. Distance calculation is also $O(1)$ [S4, unofficial].
- **Memory Complexity:** The method itself uses minimal memory, storing only the chunk size and distance parameters [S2, unofficial].
- **Strengths:** Efficiently manages large voxel worlds by limiting active memory usage to a subset of chunks [S2, unofficial]. Simple arithmetic operations make it fast to execute [S1, unofficial].
- **Limitations:** Requires careful handling of negative coordinates to avoid indexing errors [S3, unofficial]. The choice of `chunkSize` impacts both memory overhead and the frequency of chunk loading/unloading events [S2, unofficial].

## Background
Chunk-based voxel storage is a common technique in voxel engines and games to manage large worlds [S2, unofficial]. The specific coordinate mapping logic described here is found in various implementations, including the DoonEngine [S1, unofficial], voxel-engine-stackgl [S2, unofficial], and ROS-based point cloud voxelization nodes [S3, unofficial]. The distance calculation method is used in Unreal Engine-based voxel worlds [S4, unofficial].

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
[S2] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S3] zAle711/Point-Cloud-Voxelization-Node, https://github.com/zAle711/Point-Cloud-Voxelization-Node, fetched 2026-10-27 (unofficial)
[S4] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
