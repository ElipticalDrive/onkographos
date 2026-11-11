# Chunk Coordinate Normalization

Also called: Coordinate Validation, Spatial Bounds Checking, World Position Normalization, isSampleCoordInChunkBounds, isVoxelCoordInChunkBounds, isWorldPositionCanonical

## Overview
Chunk Coordinate Normalization ensures that a world-space position is represented by a canonical pair of chunk indices and local coordinates, preventing coordinate ambiguity and overflow. It differs from [Chunk Coordinate Mapping and World Bounds](7013-chunk-coordinate-mapping-and-world-bounds.md) by actively correcting non-canonical inputs (where local coordinates fall outside the chunk bounds) rather than merely converting between existing valid representations. It serves as a prerequisite for density field sampling, ensuring that subsequent lookups operate on consistent spatial data [S1, unofficial].

## How it works
**Inputs:** A `WorldPosition` structure containing `chunk` (integer indices) and `localPosition` (floating-point coordinates), along with constants `CHUNK_SIZE_METERS` (physical size of a chunk) and `CHUNK_SIZE` (voxel count per axis) [S1, unofficial].
**Outputs:** A normalized `WorldPosition` where `localPosition` components are strictly within $[0, \text{CHUNK\_SIZE\_METERS})$ and `chunk` indices are adjusted accordingly.

**Data Structures:**
*   `WorldPosition`:
    *   `chunk`: `int64_t` or `int32_t` (x, y, z)
    *   `localPosition`: `float` (x, y, z)
*   Constants:
    *   `CHUNK_SIZE_METERS`: `float`
    *   `CHUNK_SIZE`: `int`
    *   `CHUNK_SAMPLE_SIZE`: `int` (sample count per axis)

**Steps:**
1.  **Calculate Chunk Offset:** Determine how many full chunks the current `localPosition` exceeds or falls short of the canonical range.
    $$ \text{chunkOffset} = \lfloor \frac{\text{localPosition}}{\text{CHUNK\_SIZE\_METERS}} \rfloor $$
    Note: Standard floor division is used. If `localPosition` is negative, `chunkOffset` becomes negative, moving the chunk index backwards.
2.  **Update Chunk Coordinate:** Adjust the chunk index by the calculated offset.
    $$ \text{chunkCoordinate}_{\text{new}} = \text{chunkCoordinate}_{\text{old}} + \text{chunkOffset} $$
3.  **Update Local Position:** Subtract the physical distance covered by the offset from the local position to bring it back into the $[0, \text{CHUNK\_SIZE\_METERS})$ range.
    $$ \text{localPosition}_{\text{new}} = \text{localPosition}_{\text{old}} - \text{chunkOffset} \times \text{CHUNK\_SIZE\_METERS} $$
4.  **Overflow Check:** Assert that the normalized `chunkCoordinate` fits within the `int32_t` range to prevent integer overflow during subsequent calculations [S1, unofficial].
5.  **Validation:** Verify that the resulting `localPosition` components are $\ge 0.0f$ and $< \text{CHUNK\_SIZE\_METERS}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_world_position(world_position: WorldPosition, chunk_size_meters: float) -> WorldPosition:
    """
    Normalizes a WorldPosition to canonical bounds.
    """
    # Process each axis independently
    for axis in ['x', 'y', 'z']:
        local_pos = world_position.local_position[axis]
        chunk_idx = world_position.chunk[axis]
        
        # Step 1: Calculate chunk offset
        # Floor division handles negative local positions correctly
        chunk_offset = floor(local_pos / chunk_size_meters)
        
        # Step 2 & 3: Update chunk and local position
        chunk_idx += chunk_offset
        local_pos -= chunk_offset * chunk_size_meters
        
        # Update the world position object
        world_position.chunk[axis] = chunk_idx
        world_position.local_position[axis] = local_pos

    # Step 4: Overflow check
    # Assert that chunk coordinates fit in int32_t
    assert -2147483648 <= world_position.chunk.x <= 2147483647
    assert -2147483648 <= world_position.chunk.y <= 2147483647
    assert -2147483648 <= world_position.chunk.z <= 2147483647

    # Step 5: Validation
    # Ensure local position is within [0, CHUNK_SIZE_METERS)
    assert is_world_position_canonical(world_position)
    
    return world_position

def is_world_position_canonical(world_position: WorldPosition, chunk_size_meters: float) -> bool:
    """
    Checks if localPosition components are >= 0.0f and < CHUNK_SIZE_METERS.
    """
    eps = 1e-6 # Small epsilon for float comparison safety
    return (
        0.0 <= world_position.local_position.x < (chunk_size_meters - eps) and
        0.0 <= world_position.local_position.y < (chunk_size_meters - eps) and
        0.0 <= world_position.local_position.z < (chunk_size_meters - eps)
    )

def is_voxel_coord_in_chunk_bounds(voxel_coord, chunk_size: int) -> bool:
    """
    Checks voxelCoord.x, y, z < CHUNK_SIZE.
    Assumes non-negative indices.
    """
    return (
        0 <= voxel_coord.x < chunk_size and
        0 <= voxel_coord.y < chunk_size and
        0 <= voxel_coord.z < chunk_size
    )

def is_sample_coord_in_chunk_bounds(sample_coord, chunk_sample_size: int) -> bool:
    """
    Checks sampleCoord.x, y, z < CHUNK_SAMPLE_SIZE.
    """
    return (
        0 <= sample_coord.x < chunk_sample_size and
        0 <= sample_coord.y < chunk_sample_size and
        0 <= sample_coord.z < chunk_sample_size
    )
```

**Parameters:**
*   `CHUNK_SIZE_METERS`: Physical size of a chunk in world units. Typical value depends on scene scale, e.g., 1.0f or 10.0f [S1, unofficial].
*   `CHUNK_SIZE`: Voxel count per chunk axis. Not given in the sources.
*   `CHUNK_SAMPLE_SIZE`: Sample count per chunk axis. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Chunk Coordinate Overflow:** If the input `localPosition` is extremely large or small, the resulting `chunkCoordinate` may exceed `int32_t` limits. This is handled by an assertion against `std::numeric_limits<int32_t>::min()` and `max()` [S1, unofficial].
*   **Non-Canonical Input:** If `localPosition` components are outside $[0, \text{CHUNK\_SIZE\_METERS})$, normalization adjusts chunk coordinates. The final assert ensures the result is canonical [S1, unofficial].
*   **Floating Point Precision:** Direct comparison of `localPosition` against `CHUNK_SIZE_METERS` should account for floating-point errors. The pseudocode uses an epsilon for safety [G].

## Subtopics

### World Position Construction
Initializes a `WorldPosition` from a chunk coordinate and a local position. It calls `normalize_world_position` for each axis (x, y, z) sequentially. The resulting position must pass `is_world_position_canonical`. If the input `localPosition` is non-canonical, the normalization step adjusts the chunk coordinates, and the final assertion ensures the state is canonical [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per axis. The operation involves basic arithmetic (division, multiplication, addition) and comparisons.
*   **Memory Complexity:** $O(1)$. No additional data structures are allocated; it modifies the input structure in place or returns a new one.
*   **Strengths:** Ensures consistent spatial indexing, preventing artifacts in density field sampling and surface extraction. Prevents integer overflow in chunk indexing.
*   **Limitations:** Requires floating-point division, which is slightly more expensive than integer arithmetic. The assertion on `int32_t` range limits the maximum world extent to approximately $\pm 2 \times 10^9$ chunks.
*   **When to use:** Before any density field sampling or spatial query that relies on chunk-based indexing.
*   **When not to use:** When working with purely local chunk coordinates that are already guaranteed to be canonical.

## Background
Originated in the PlaneVoxelLab project by aaron-eugene [S1, unofficial]. The method is part of a dense grid and memory layout system for volume rendering, specifically within chunk-based voxel data structures. It ensures that world-space positions are correctly mapped to chunk-local coordinates for efficient density field evaluation [S1, unofficial].

## Sources
[S1] aaron-eugene/PlaneVoxelLab, https://github.com/aaron-eugene/PlaneVoxelLab, fetched 2026-10-27 (unofficial)
