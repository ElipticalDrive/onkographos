# Voxel Block Placement with Face Culling

Also called: Chunk Block Toggle, Dynamic Voxel Meshing

## Overview
Voxel Block Placement with Face Culling is a technique for dynamically updating a voxel chunk's internal state and face visibility when blocks are placed or removed, using neighbor checks to toggle shared faces without regenerating the entire mesh [S1, unofficial]. It differs from its sibling [Greedy face merging](8935-greedy-face-merging.md) by maintaining per-block visibility flags rather than merging adjacent quads into larger primitives, and from [Blocky meshing](31-blocky-meshing.md) generally by focusing on incremental state updates rather than batch mesh generation.

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel chunk containing a flat array of block data, a block position $(x, y, z)$, and a block ID.
*   **Output:** Updated chunk state reflecting the presence/absence of the block and the visibility of adjacent faces.

**Data Structures:**
The chunk stores block data in a flat array: `uint32_t data[CHUNK_SIZE * CHUNK_SIZE * CHUNK_SIZE]` where `CHUNK_SIZE = 16` [S1, unofficial].
*   **Total Size:** Each element is 4096 bytes total (Note: `uint32_t` is 4 bytes; the source text "4096 bytes total" likely refers to the entire chunk size $16^3 \times 4 = 4096$ bytes, or is a typo for 4 bytes per element. Given `uint32_t`, the element size is 4 bytes) [S1, unofficial].
*   **Index Mapping:** `pos = x | (y << 4) | (z << 8)` [S1, unofficial].
*   **Block ID:** Occupies the lower 6 bits (mask `0x3F`). An empty block is `0`. A block with ID `id` is stored as `0x3F000000 | id` (Note: The source text `0x3F000000 | id` places the ID in the upper bits, but the description says "lower 6 bits". The mask `0x3F` implies lower bits. The value `0x3F000000` suggests upper bits. We follow the explicit storage formula `0x3F000000 | id` as written, but note the contradiction with "lower 6 bits" and mask `0x3F`. Standard interpretation of "lower 6 bits" would be `id & 0x3F`. The source explicitly writes `0x3F000000 | id`. We use the explicit formula) [S1, unofficial].
*   **Face Visibility:** Remaining bits track face visibility via `Block::toggleBlockFace` [S1, unofficial].

**Algorithm Steps:**

1.  **Place Block:**
    1.  Check bounds. If `data[pos] == 0` and `id > 0`, set `data[pos] = 0x3F000000 | id` and increment `blockCount` [S1, unofficial].
    2.  For each of the 6 neighbors (directions $\pm x, \pm y, \pm z$):
        a.  Call `toggleFace` on the neighbor.
        b.  If the neighbor exists (within bounds), call `toggleFace` on the current block's corresponding opposite face [S1, unofficial].

2.  **Remove Block:**
    1.  Check bounds. If `data[pos] > 0` and `id == 0` (Note: Source says `id == 0` for removal check, but `data[pos]` stores ID. This likely implies checking if the block at `pos` is the one being removed, or if the slot is empty. The source text is `If data[pos] > 0 and id == 0`. This is ambiguous. Usually, removal checks if `data[pos]` matches the ID of the block being removed. We follow the source text literally: `data[pos] > 0` and `id == 0`), set `data[pos] = 0` and decrement `blockCount` [S1, unofficial].
    2.  Call `toggleFace` on all 6 neighbors unconditionally to update their face states [S1, unofficial].

3.  **Neighbor Face Toggle:**
    1.  If neighbor coordinates are within chunk bounds, compute neighbor `pos` [S1, unofficial].
    2.  If neighbor block exists (`data[neighbor_pos] > 0`), call `Block::toggleBlockFace(data + neighbor_pos, side)` [S1, unofficial].
    3.  Return true if toggle occurred, false otherwise [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
CHUNK_SIZE = 16
MASK_ID = 0x3F
MASK_VISIBILITY = ~MASK_ID # Remaining bits

def index(x, y, z):
    return x | (y << 4) | (z << 8)

def is_in_bounds(x, y, z):
    return 0 <= x < CHUNK_SIZE and 0 <= y < CHUNK_SIZE and 0 <= z < CHUNK_SIZE

def toggle_block_face(block_data_ptr, side):
    # Updates the visibility bit for the given side on the block.
    # Implementation details of bit manipulation are not provided in source [G].
    pass

def place_block(chunk_data, x, y, z, block_id):
    pos = index(x, y, z)
    if not is_in_bounds(x, y, z):
        return
    
    current_val = chunk_data[pos]
    # Check if empty and ID is valid
    if current_val == 0 and block_id > 0:
        chunk_data[pos] = 0x3F000000 | block_id
        # Increment blockCount [G]
        
        # Directions: +x, -x, +y, -y, +z, -z
        directions = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
        
        for dx, dy, dz in directions:
            nx, ny, nz = x + dx, y + dy, z + dz
            if is_in_bounds(nx, ny, nz):
                n_pos = index(nx, ny, nz)
                # Toggle face on neighbor
                toggle_block_face(chunk_data[n_pos], get_opposite_side(dx, dy, dz))
                # Toggle face on current block (opposite side)
                toggle_block_face(chunk_data[pos], get_side(dx, dy, dz))

def remove_block(chunk_data, x, y, z, block_id):
    pos = index(x, y, z)
    if not is_in_bounds(x, y, z):
        return
    
    current_val = chunk_data[pos]
    # Source condition: data[pos] > 0 and id == 0
    if current_val > 0 and block_id == 0:
        chunk_data[pos] = 0
        # Decrement blockCount [G]
        
        directions = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
        for dx, dy, dz in directions:
            nx, ny, nz = x + dx, y + dy, z + dz
            if is_in_bounds(nx, ny, nz):
                n_pos = index(nx, ny, nz)
                toggle_block_face(chunk_data[n_pos], get_opposite_side(dx, dy, dz))

def get_side(dx, dy, dz):
    # Maps direction vector to face side enum
    pass

def get_opposite_side(dx, dy, dz):
    # Maps direction vector to opposite face side enum
    pass
```

**Edge Cases and Pitfalls:**
*   **Boundary Checks:** Prevent out-of-bounds memory access when toggling faces at chunk edges. If a neighbor is outside the chunk, `toggleFace` returns false, correctly skipping the face update for that direction [S1, unofficial].
*   **Data Layout Ambiguity:** The source states the ID occupies the "lower 6 bits" with mask `0x3F`, but the storage formula is `0x3F000000 | id`. This places the ID in the upper bits. Implementers should verify the intended bit layout [S1, unofficial].

## Subtopics

### [Static Mesh Caching](1138-static-mesh-caching.md)
Static Mesh Caching reduces draw calls and GPU data upload frequency by aggregating all voxel geometry into a single `Mesh` object. It differs from its parent [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by ignoring shared face optimization; instead of toggling visibility, it rebuilds the full geometry of every visible voxel into one buffer. It differs from [Dynamic Block Mesh Management](3574-dynamic-block-mesh-management.md) by maintaining a single global mesh rather than managing multiple independent mesh objects.

### [Dynamic Block Mesh Management](3574-dynamic-block-mesh-management.md)
Dynamic Block Mesh Management is a technique for maintaining a dynamic list of block indices organized by spatial coordinates, allowing for the efficient addition and removal of individual block faces without rebuilding the entire mesh. It solves the problem of high computational cost associated with regenerating voxel meshes upon every block placement or removal by updating only the affected local geometry.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per block placement/removal, as it only updates the block itself and its 6 immediate neighbors.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of blocks in the chunk. The flat array size is fixed at $16^3$ elements.
*   **Strengths:** Incremental updates avoid full mesh regeneration, making it suitable for dynamic voxel environments.
*   **Limitations:** Does not merge adjacent faces, resulting in higher polygon counts compared to greedy meshing.
*   **When to use:** When blocks are frequently added/removed and dynamic updates are required.
*   **When not to use:** When static geometry or maximum rendering performance (via face merging) is prioritized over update speed.

## Background
Originating from the Shiranuit/VoxelEngine-0 project, this method is part of a broader effort in efficient voxel rendering [S1, unofficial]. It builds on the principles of [Blocky meshing](31-blocky-meshing.md) by focusing on the management of block visibility states.

## Sources
[S1] Shiranuit/VoxelEngine-0, https://github.com/Shiranuit/VoxelEngine-0, fetched 2026-10-27 (unofficial)
