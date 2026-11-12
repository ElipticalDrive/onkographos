# Chunk Coordinate Mapping and World Bounds

Also called: Chunk-based Voxelization and Position Mapping, Chunk-to-World Transform

## Overview
This method decomposes a global voxel position into map-space chunk coordinates and chunk-local voxel coordinates, enabling efficient memory management by dividing the voxel world into fixed-size chunks [S1, S2, unofficial]. It converts between chunk grid indices and world coordinates and calculates the total world width covered by the chunk system [S2, unofficial]. Unlike [Chunk Coordinate Normalization](12959-chunk-coordinate-normalization.md), which actively corrects non-canonical inputs to prevent overflow, this method focuses on the primary translation between valid world, chunk, and local representations [S1, unofficial].

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
- `pos`: `int3` (or equivalent 3-component integer vector).
- `mapPos`: `int3`.
- `chunkPos`: `int3`.
- `chunkSize`: `int` or `float` (depending on implementation precision).
- `chunkDistance`: `int`.

**Steps:**

1. **Calculate World Bounds (Optional but common):**
   Compute the total world width covered by the chunk system using the formula:
   $$ \text{worldWidth} = \text{chunkSize} \times 2 \times \text{chunkDistance} $$
   This defines the active simulation or rendering volume centered around the origin or player [S2, unofficial].

2. **Compute Chunk Index (Map Position):**
   Determine which chunk contains the global voxel position. This is typically achieved via integer division:
   $$ \text{mapPos}_i = \lfloor \text{pos}_i / \text{chunkSize} \rfloor $$
   where $i \in \{x, y, z\}$ [S1, unofficial]. Note that for negative coordinates, standard integer division behavior (truncation toward zero vs. floor) must be handled consistently to ensure correct chunk indexing [S3, unofficial].

3. **Compute Local Position (Chunk Position):**
   Determine the voxel's offset within the identified chunk. This is typically achieved via the modulo operator:
   $$ \text{chunkPos}_i = \text{pos}_i \pmod{\text{chunkSize}} $$
   [S1, unofficial].

4. **Handle Negative Coordinates (Mirroring/Offset):**
   If the global coordinate is negative, special handling is required to maintain consistent chunk indexing and avoid boundary errors [S3, unofficial]. One approach involves calculating a fractional part and inverting the local position:
   - Calculate `frac_pos` based on the absolute coordinate.
   - If `pos_i < 0`, adjust `local_pos` (chunkPos) to reflect the mirrored nature within the chunk structure [S3, unofficial].
   - Specifically, some implementations use:
     $$ \text{local\_pos} = \begin{cases} \text{frac\_pos} & \text{if } \text{sign\_positive} \\ \text{chunkSize} - 1 - \text{frac\_pos} & \text{otherwise} \end{cases} $$
     [S3, unofficial].

5. **Convert to World Coordinates (Inverse Mapping):**
   To convert chunk indices back to world coordinates, multiply the chunk position by the chunk size:
   $$ \text{worldCoord}_i = \text{mapPos}_i \times \text{chunkSize} $$
   [S2, unofficial]. For precise floating-point world positions including local offsets, combine the chunk base and local position:
   $$ \text{final\_coord}_i = \begin{cases} \text{chunk\_pos}_i + \frac{\text{local\_pos}_i}{100.0} & \text{if } \text{sign\_positive} \\ \text{chunk\_pos}_i - \frac{\text{local\_pos}_i}{100.0} & \text{otherwise} \end{cases} $$
   [S3, unofficial]. Note: The divisor `100.0` in [S3, unofficial] appears specific to a particular implementation's internal scaling (possibly for sub-voxel precision or ROS parameter units) and may not be universal [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunk_coordinate_mapping(pos: Vector3i, chunk_size: int) -> tuple[Vector3i, Vector3i]:
    """
    Decomposes a global voxel position into chunk coordinates and local voxel coordinates.
    
    Args:
        pos: Global voxel position (int3).
        chunk_size: Number of voxels per chunk axis (int).
        
    Returns:
        map_pos: Chunk index in the global grid (int3).
        local_pos: Local voxel position within the chunk (int3).
    """
    map_pos = Vector3i()
    local_pos = Vector3i()
    
    for i in range(3):
        # Step 2: Compute Chunk Index
        # Using floor division to handle negative coordinates correctly
        # In many languages, integer division truncates toward zero, so explicit floor is needed for negatives
        if pos[i] >= 0:
            map_pos[i] = pos[i] // chunk_size
        else:
            # Floor division for negative numbers
            map_pos[i] = -((-pos[i] + chunk_size - 1) // chunk_size)
            
        # Step 3: Compute Local Position
        # Standard modulo
        local_pos[i] = pos[i] % chunk_size
        
        # Step 4: Handle Negative Coordinate Mirroring (if required by specific implementation)
        # Some systems invert local coordinates for negative chunks to maintain consistent 
        # internal indexing or mirroring logic [S3, unofficial]. This is implementation-specific.
        # If mirroring is active:
        # if pos[i] < 0:
        #     local_pos[i] = chunk_size - 1 - local_pos[i]
        
    return map_pos, local_pos

def world_to_chunk(world_pos: Vector3f, chunk_size: float) -> tuple[Vector3i, Vector3f]:
    """
    Converts world-space floating point coordinates to chunk indices and local offsets.
    
    Args:
        world_pos: World space position (float3).
        chunk_size: Size of a chunk in world units (float).
        
    Returns:
        chunk_idx: Chunk grid index (int3).
        local_offset: Local position within the chunk (float3).
    """
    chunk_idx = Vector3i()
    local_offset = Vector3f()
    
    for i in range(3):
        # Compute chunk index
        # Floor is used to determine the chunk containing the point
        chunk_idx[i] = int(math.floor(world_pos[i] / chunk_size))
        
        # Compute local offset
        # Subtract the chunk's base world position to get the local coordinate
        chunk_base = chunk_idx[i] * chunk_size
        local_offset[i] = world_pos[i] - chunk_base
        
    return chunk_idx, local_offset
```

**Parameters:**
- `chunkSize`: Typical values are 16 or 32 voxels, balancing memory overhead per chunk against the granularity of loading/unloading [S1, S2, unofficial].
- `chunkDistance`: Typically 2 or more chunks, defining the radius of the active area around the viewer [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Negative Coordinates:** Standard integer division and modulo operations in many programming languages truncate toward zero, which can lead to incorrect chunk indices for negative world coordinates (e.g., `-1 // 16` might be `0` instead of `-1`). Explicit floor division or offset adjustments are required [S3, unofficial].
- **Boundary Errors:** Failing to handle the inversion or offset for negative coordinates can cause discontinuities in data access or rendering artifacts at chunk boundaries [S3, unofficial].
- **Precision:** When converting between integer voxel coordinates and floating-point world coordinates, precision loss can occur if not handled carefully, especially with large world coordinates [S3, unofficial].

## Subtopics

### [Chunk Coordinate Normalization](12959-chunk-coordinate-normalization.md)
Chunk Coordinate Normalization ensures that a world-space position is represented by a canonical pair of chunk indices and local coordinates, preventing coordinate ambiguity and overflow. It differs from Chunk Coordinate Mapping and World Bounds by actively correcting non-canonical inputs (where local coordinates fall outside the chunk bounds) rather than merely converting between existing valid representations. It serves as a prerequisite for density field sampling, ensuring that subsequent lookups operate on consistent spatial data.

### Chunk Distance Calculation
Calculates the squared distance between the viewer position and the center of a chunk in world units [S4, unofficial]. This is used to determine if a chunk should be loaded or unloaded based on the viewer's proximity [S4, unofficial]. The calculation involves determining the chunk's center in world space and then computing the Euclidean distance (or squared distance for efficiency) to the viewer's position [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Coordinate mapping is $O(1)$ per axis, involving basic arithmetic operations (division, modulo) [S1, S2, unofficial]. Distance calculation is also $O(1)$ [S4, unofficial].
- **Memory Complexity:** The method itself uses minimal memory, storing only the chunk size and distance parameters [S2, unofficial]. The memory savings come from the chunk-based storage structure it enables, which only loads chunks within `chunkDistance` [S2, unofficial].
- **Strengths:** Efficiently manages large voxel worlds by limiting active memory usage to a subset of chunks [S2, unofficial]. Simple arithmetic operations make it fast to execute [S1, unofficial].
- **Limitations:** Requires careful handling of negative coordinates to avoid indexing errors [S3, unofficial]. The choice of `chunkSize` impacts both memory overhead and the frequency of chunk loading/unloading events [S2, unofficial].
- **When to Use:** Ideal for large-scale voxel worlds where memory is a constraint and the viewer's position is dynamic [S2, unofficial].

## Background
Chunk-based voxel storage is a common technique in voxel engines and games to manage large worlds [S2, unofficial]. The specific coordinate mapping logic described here is found in various implementations, including the DoonEngine [S1, unofficial], voxel-engine-stackgl [S2, unofficial], and ROS-based point cloud voxelization nodes [S3, unofficial]. The distance calculation method is used in Unreal Engine-based voxel worlds [S4, unofficial].

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
[S2] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S3] zAle711/Point-Cloud-Voxelization-Node, https://github.com/zAle711/Point-Cloud-Voxelization-Node, fetched 2026-10-27 (unofficial)
[S4] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
