# Morton Code Incremental Traversal

Also called: VolumeSampler traversal, Z-order curve traversal

## Overview
An algorithm for iterating through a 3D voxel volume in Morton (Z-order) curve order using precomputed delta values to update the linear index incrementally, avoiding expensive coordinate-to-index conversions except at chunk boundaries [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel volume organized into fixed-size chunks, a current global position $(X, Y, Z)$, and precomputed delta values for traversal directions [S1, unofficial].
*   **Output:** The voxel data at the current position and an updated internal state for the next traversal step [S1, unofficial].

**Data Structures:**
1.  **Chunk State:**
    *   `CurrentChunk`: Pointer or handle to the active chunk data [S1, unofficial].
    *   `CurrentVoxelIndex`: The linear Morton index within the current chunk [S1, unofficial].
    *   `XPosInChunk`, `YPosInChunk`, `ZPosInChunk`: Local coordinates within the chunk [S1, unofficial].
    *   `XPosInVolume`, `YPosInVolume`, `ZPosInVolume`: Global coordinates [S1, unofficial].
2.  **Lookup Tables:**
    *   `deltaX`, `deltaY`, `deltaZ`: Static constant lookup tables storing the integer offset to add to `CurrentVoxelIndex` when moving one step in the positive X, Y, or Z direction within a chunk of side length 16 [S1, unofficial].
    *   `morton256_x`, `morton256_y`, `morton256_z`: Precomputed bit-interleaving tables used for initial index calculation [S1, unofficial].

**Parameters:**
*   `ChunkSideLengthMinusOne`: The maximum coordinate value within a chunk. For a standard $16^3$ chunk, this is 15. Used for boundary checks [S1, unofficial].

**Numbered Steps:**

1.  **Initialization (`SetPosition`):**
    When the global position changes or at startup [S1, unofficial]:
    a. Calculate chunk coordinates:
       $$xChunk = XPosInVolume \gg sideLengthPower$$
       $$yChunk = YPosInVolume \gg sideLengthPower$$
       $$zChunk = ZPosInVolume \gg sideLengthPower$$ [S1, unofficial].
    b. Calculate intra-chunk coordinates:
       $$XPosInChunk = XPosInVolume - (xChunk \ll sideLengthPower)$$
       $$YPosInChunk = YPosInVolume - (yChunk \ll sideLengthPower)$$
       $$ZPosInChunk = ZPosInVolume - (zChunk \ll sideLengthPower)$$ [S1, unofficial].
    c. Compute initial voxel index using Morton bit-interleaving:
       $$CurrentVoxelIndex = morton256_x[XPosInChunk] \ | \ morton256_y[YPosInChunk] \ | \ morton256_z[ZPosInChunk]$$ [S1, unofficial].
    d. Retrieve the chunk handle for $(xChunk, yChunk, zChunk)$ [G].

2.  **Incremental Traversal:**
    To move to an immediate neighbour (e.g., positive Y) [S1, unofficial]:
    a. Update global coordinate: $YPosInVolume \leftarrow YPosInVolume + 1$ [S1, unofficial].
    b. Check boundary condition: If $YPosInChunk < \text{ChunkSideLengthMinusOne}$ [S1, unofficial]:
       i. Update local coordinate: $YPosInChunk \leftarrow YPosInChunk + 1$ [S1, unofficial].
       ii. Update linear index: $CurrentVoxelIndex \leftarrow CurrentVoxelIndex + \text{POS\_Y\_DELTA}$ [S1, unofficial].
    c. If boundary reached ($YPosInChunk == \text{ChunkSideLengthMinusOne}$):
       i. Call `SetPosition` to recompute chunk and index from global coordinates [S1, unofficial].

3.  **Voxel Access:**
    *   If `CurrentChunk` is valid, return `CurrentChunk->GetDataAtIndex(CurrentVoxelIndex)` [S1, unofficial].
    *   If `CurrentChunk` is null, fallback to `Volume->GetVoxelByCoordinates(XPosInVolume, YPosInVolume, ZPosInVolume)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class MortonSampler:
    def __init__(self, volume, chunk_side_length=16):
        self.volume = volume
        self.chunk_side_length = chunk_side_length
        self.chunk_side_length_minus_one = chunk_side_length - 1
        self.side_length_power = 4  # log2(16)
        
        # Precomputed tables (loaded from source)
        self.delta_x = [0] * 256
        self.delta_y = [0] * 256
        self.delta_z = [0] * 256
        self.morton256_x = [0] * 256
        self.morton256_y = [0] * 256
        self.morton256_z = [0] * 256
        
        self.current_chunk = None
        self.current_voxel_index = 0
        self.x_pos_in_chunk = 0
        self.y_pos_in_chunk = 0
        self.z_pos_in_chunk = 0
        self.x_pos_in_volume = 0
        self.y_pos_in_volume = 0
        self.z_pos_in_volume = 0

    def set_position(self, x, y, z):
        self.x_pos_in_volume = x
        self.y_pos_in_volume = y
        self.z_pos_in_volume = z
        
        # Calculate chunk coordinates
        x_chunk = x >> self.side_length_power
        y_chunk = y >> self.side_length_power
        z_chunk = z >> self.side_length_power
        
        # Calculate intra-chunk coordinates
        self.x_pos_in_chunk = x - (x_chunk << self.side_length_power)
        self.y_pos_in_chunk = y - (y_chunk << self.side_length_power)
        self.z_pos_in_chunk = z - (z_chunk << self.side_length_power)
        
        # Compute initial voxel index
        self.current_voxel_index = (
            self.morton256_x[self.x_pos_in_chunk] |
            self.morton256_y[self.y_pos_in_chunk] |
            self.morton256_z[self.z_pos_in_chunk]
        )
        
        # Get chunk handle (implementation-specific)
        self.current_chunk = self.volume.get_chunk(x_chunk, y_chunk, z_chunk)

    def move_positive_y(self):
        self.y_pos_in_volume += 1
        
        if self.y_pos_in_chunk < self.chunk_side_length_minus_one:
            self.y_pos_in_chunk += 1
            # delta_y[0] holds the fixed stride for the +Y direction within the chunk
            self.current_voxel_index += self.delta_y[0]  
        else:
            self.set_position(self.x_pos_in_volume, self.y_pos_in_volume, self.z_pos_in_volume)

    def get_voxel(self):
        if self.current_chunk is None:
            return self.volume.get_voxel_by_coordinates(
                self.x_pos_in_volume, 
                self.y_pos_in_volume, 
                self.z_pos_in_volume
            )
        else:
            return self.current_chunk.get_data_at_index(self.current_voxel_index)
```

**Edge Cases and Pitfalls:**
*   **Chunk Boundary Crossing:** When moving across a chunk boundary (e.g., $XPosInChunk$ becomes 16 or -1), the incremental delta update is invalid because the Morton curve resets. The algorithm must fall back to full coordinate-to-index conversion via `SetPosition` [S1, unofficial].
*   **Null Chunk:** If the chunk pointer is null, the sampler must fallback to global coordinate lookup [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Found in the PolyVox library implementation within the VoxelGenerator repository [S1, unofficial].

## Sources
[S1] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
