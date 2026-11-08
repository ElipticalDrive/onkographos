# Chunk Coordinate Mapping and World Bounds

Also called: chunk-to-world transform

## Overview
Chunk Coordinate Mapping and World Bounds decomposes a global voxel position into map-space chunk coordinates and chunk-local voxel coordinates, while simultaneously converting between chunk grid indices and world coordinates to determine the total world width covered by the chunk system [S1, S2, unofficial]. It serves as the fundamental arithmetic bridge between the logical chunk grid and the continuous world space, enabling efficient spatial partitioning [S1, S2, unofficial]. Unlike [Voxel World Storage](3984-voxel-world-storage.md) which may use flat arrays, this method explicitly manages the hierarchical translation required for [Chunk-Based World Storage](2858-chunk-based-world-storage.md) to function, distinguishing it from sibling methods by focusing on the coordinate transformation logic itself rather than the loading strategy [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Global voxel position `pos` (integer vector, e.g., `DNivec3` or `[x, y, z]`), chunk size `chunkSize` (integer), and optionally chunk distance `chunkDistance` (integer) for bounds calculation [S1, S2, unofficial].
*   **Outputs:** Chunk position `mapPos` (integer vector representing the chunk index in the grid) and local position `chunkPos` (integer vector representing the voxel index within the chunk) [S1, unofficial]. World coordinates `worldPos` (float vector) and total world bounds dimensions [S2, unofficial].

**Data Structures:**
*   `pos`: A 3D integer vector representing coordinates in the global voxel space.
*   `chunkSize`: A scalar integer defining the number of voxels along one axis per chunk. Typical values are 16 or 32, balancing memory overhead against culling granularity [S2, unofficial].
*   `chunkDistance`: A scalar integer defining the radius of loaded chunks around the viewer/player in chunk indices [S2, unofficial].

**Steps:**

1.  **Calculate Chunk Bounds (World Width):**
    Determine the total extent of the managed world. The world width along one axis is calculated as:
    $$ \text{WorldWidth} = \text{chunkSize} \times 2 \times \text{chunkDistance} $$
    This defines the active simulation or rendering volume centered around the origin or player [S2, unofficial].

2.  **Decompose Global Position:**
    Given a global voxel position `pos`, compute the chunk index and local offset.
    *   **Chunk Index (`mapPos`):**
        $$ \text{mapPos} = \lfloor \text{pos} / \text{chunkSize} \rfloor $$
        This integer division yields the chunk coordinates in the grid [S1, unofficial].
    *   **Local Position (`chunkPos`):**
        $$ \text{chunkPos} = \text{pos} \pmod{\text{chunkSize}} $$
        This modulo operation yields the voxel's position within the identified chunk [S1, unofficial].

3.  **Convert Chunk to World Coordinates:**
    To convert a chunk grid index `chunkIdx` (integer vector) to world coordinates `worldPos` (float vector):
    $$ \text{worldPos} = \text{chunkIdx} \times \text{chunkSize} $$
    This maps the chunk's origin (typically its minimum corner) to world space [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompose_position(pos: tuple[int, int, int], chunk_size: int) -> tuple[tuple[int, int, int], tuple[int, int, int]]:
    """
    Decomposes a global voxel position into chunk coordinates and local voxel coordinates.
    
    Args:
        pos: Global voxel position (x, y, z).
        chunk_size: Number of voxels per chunk axis.
        
    Returns:
        mapPos: Chunk coordinates in the grid.
        chunkPos: Local voxel coordinates within the chunk.
    """
    # Integer division for chunk index
    map_x = pos[0] // chunk_size
    map_y = pos[1] // chunk_size
    map_z = pos[2] // chunk_size
    
    # Modulo for local position
    local_x = pos[0] % chunk_size
    local_y = pos[1] % chunk_size
    local_z = pos[2] % chunk_size
    
    return (map_x, map_y, map_z), (local_x, local_y, local_z)

def chunk_to_world(chunk_idx: tuple[int, int, int], chunk_size: int) -> tuple[float, float, float]:
    """
    Converts chunk grid indices to world coordinates.
    
    Args:
        chunk_idx: Chunk coordinates (x, y, z).
        chunk_size: Number of voxels per chunk axis.
        
    Returns:
        world_pos: World coordinates of the chunk's origin.
    """
    return (chunk_idx[0] * chunk_size, 
            chunk_idx[1] * chunk_size, 
            chunk_idx[2] * chunk_size)

def calculate_world_bounds(chunk_size: int, chunk_distance: int) -> int:
    """
    Calculates the total world width covered by the chunk system.
    
    Args:
        chunk_size: Number of voxels per chunk axis.
        chunk_distance: Radius of loaded chunks in chunk indices.
        
    Returns:
        world_width: Total width of the world along one axis.
    """
    return chunk_size * 2 * chunk_distance
```

**Edge Cases and Pitfalls:**
*   **Negative Coordinates:** The modulo operator behavior for negative numbers varies by language (e.g., Python returns positive remainders, C++/Java may return negative). Implementers must ensure `chunkPos` is correctly handled for negative world coordinates, often by adjusting the division logic or using floor division consistently [G].
*   **Boundary Lag:** Setting `chunkDistance` equal to `removeDistance` (if used for unloading) causes lag when crossing chunk boundaries due to immediate reload cycles. `removeDistance` should typically be `chunkDistance + 1` [S2, unofficial].
*   **Precision:** World coordinates are typically floats, while chunk indices are integers. Multiplication by `chunkSize` is exact for integers, but subsequent floating-point operations (like distance calculations) may introduce minor precision errors [S2, S3, unofficial].

## Subtopics

### Chunk Distance Calculation
Calculates the distance between the viewer position and the center of a chunk to determine loading/unloading eligibility. It computes the squared distance first for efficiency, then optionally the square root for linear distance [S3, unofficial].
*   **Definition:** Computes the distance from the viewer's world position to the geometric center of a specific chunk [S3, unofficial].
*   **Steps:**
    1.  Calculate the chunk's center in world units: $\text{ChunkCenter} = \text{ChunkToWorldCenter}(\text{ChunkCoord}, \text{ChunkSize}, \text{VoxelSize})$.
    2.  Compute squared Euclidean distance: $D^2 = \|\text{ViewerPos} - \text{ChunkCenter}\|^2$ [S3, unofficial].
    3.  Return $D^2$ for comparison against a squared threshold, or $\sqrt{D^2}$ for linear distance [S3, unofficial].
*   **When to choose:** Use this when implementing dynamic chunk loading based on viewer proximity rather than a fixed radius [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for coordinate decomposition and conversion. The operations involve basic integer arithmetic (division, modulo) and floating-point multiplication [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space. No additional data structures are required beyond the input coordinates and constants [S1, S2, unofficial].
*   **Strengths:** Extremely fast, deterministic, and cache-friendly. It enables efficient spatial hashing and chunk lookup without traversing complex tree structures [S1, S2, unofficial].
*   **Limitations:** Does not handle chunk persistence, loading, or unloading logic itself; it only provides the coordinate mapping. Requires careful handling of negative coordinates and boundary conditions [S2, unofficial].
*   **When to use:** Whenever voxel positions need to be mapped to or from chunk indices or world space. Essential for any chunk-based voxel system [S1, S2, unofficial].

## Background
Chunk coordinate mapping is a foundational technique in voxel engine development, emerging with the rise of Minecraft-like games and procedural terrain generation. It allows engines to manage vast worlds by dividing them into manageable, fixed-size blocks (chunks) [S1, S2, unofficial]. The specific implementations cited here reflect common patterns in open-source voxel engines, where efficient coordinate translation is critical for performance [S1, S2, S3, unofficial].

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
[S2] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S3] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
