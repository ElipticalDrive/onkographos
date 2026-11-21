# Voxel Block Placement with Face Culling

Also called: Chunk Block Toggle, Dynamic Voxel Meshing

## Overview
Voxel Block Placement with Face Culling is a technique for dynamically updating a voxel chunk's internal state and face visibility when blocks are placed or removed, using neighbor checks to toggle shared faces without regenerating the entire mesh [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel chunk containing a flat array of block data, a block position $(x, y, z)$, and a block ID [S1, unofficial].
*   **Output:** Updated chunk state reflecting the presence/absence of the block and the visibility of adjacent faces [S1, unofficial].

**Data Structures:**
The chunk stores block data in a flat array: `uint32_t data[CHUNK_SIZE * CHUNK_SIZE * CHUNK_SIZE]` where `CHUNK_SIZE = 16` [S1, unofficial].
*   **Element Size:** Each element is 4 bytes; the source's "4096 bytes total" refers to the full chunk size $16^3 \times 4$ [S1, unofficial].
*   **Index Mapping:** `pos = x | (y << 4) | (z << 8)` [S1, unofficial].
*   **Block ID:** Occupies the lower 6 bits (mask `0x3F`). The source specifies storage as `0x3F000000 | id`, which places the ID in the upper bits, contradicting the "lower 6 bits" description [S1, unofficial].
*   **Face Visibility:** Remaining bits track face visibility via `Block::toggleBlockFace` [S1, unofficial].

**Algorithm Steps:**
1.  **Place Block:**
    1.  Check bounds. If `data[pos] == 0` and `id > 0`, set `data[pos] = 0x3F000000 | id` and increment `blockCount` [S1, unofficial].
    2.  For each of the 6 neighbors (directions $\pm x, \pm y, \pm z$):
        a.  Call `toggleFace` on the neighbor.
        b.  If the neighbor exists (within bounds), call `toggleFace` on the current block's corresponding opposite face [S1, unofficial].
2.  **Remove Block:**
    1.  Check bounds. If `data[pos] > 0` and `id == 0`, set `data[pos] = 0` and decrement `blockCount` [S1, unofficial].
    2.  Call `toggleFace` on all 6 neighbors to update their face states [S1, unofficial].
3.  **Neighbor Face Toggle:**
    1.  If neighbor coordinates are within chunk bounds, compute neighbor `pos` [S1, unofficial].
    2.  If neighbor block exists (`data[neighbor_pos] > 0`), call `Block::toggleBlockFace(data + neighbor_pos, side)` [S1, unofficial].
    3.  Return true if toggle occurred, false otherwise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
CHUNK_SIZE = 16

def index(x, y, z):
    return x | (y << 4) | (z << 8)

def is_in_bounds(x, y, z):
    return 0 <= x < CHUNK_SIZE and 0 <= y < CHUNK_SIZE and 0 <= z < CHUNK_SIZE

def toggle_block_face(block_data, side):
    # Toggles the visibility bit for the given side.
    # The exact side-to-bit mapping is not specified in the source [G].
    block_data ^= (1 << side)

def place_block(chunk_data, x, y, z, block_id):
    pos = index(x, y, z)
    if not is_in_bounds(x, y, z):
        return

    current_val = chunk_data[pos]
    if current_val == 0 and block_id > 0:
        chunk_data[pos] = 0x3F000000 | block_id
        # Increment blockCount [G]

        directions = [(1, 0, 0), (-1, 0, 0), (0, 1, 0), (0, -1, 0), (0, 0, 1), (0, 0, -1)]
        for dx, dy, dz in directions:
            nx, ny, nz = x + dx, y + dy, z + dz
            if is_in_bounds(nx, ny, nz):
                n_pos = index(nx, ny, nz)
                # Toggle face on neighbor
                toggle_block_face(chunk_data[n_pos], get_side(dx, dy, dz))
                # Toggle face on current block (opposite side)
                toggle_block_face(chunk_data[pos], get_opposite_side(dx, dy, dz))

def remove_block(chunk_data, x, y, z, block_id):
    pos = index(x, y, z)
    if not is_in_bounds(x, y, z):
        return

    current_val = chunk_data[pos]
    if current_val > 0 and block_id == 0:
        chunk_data[pos] = 0
        # Decrement blockCount [G]

        directions = [(1, 0, 0), (-1, 0, 0), (0, 1, 0), (0, -1, 0), (0, 0, 1), (0, 0, -1)]
        for dx, dy, dz in directions:
            nx, ny, nz = x + dx, y + dy, z + dz
            if is_in_bounds(nx, ny, nz):
                n_pos = index(nx, ny, nz)
                toggle_block_face(chunk_data[n_pos], get_opposite_side(dx, dy, dz))

def get_side(dx, dy, dz):
    # Maps direction vector to face side enum 0-5 [G]
    return 0

def get_opposite_side(dx, dy, dz):
    # Maps direction vector to opposite face side enum 0-5 [G]
    return 0
```

**Edge Cases and Pitfalls:**
*   **Boundary Checks:** Prevent out-of-bounds memory access when toggling faces at chunk edges. If a neighbor is outside the chunk, `toggleFace` returns false, correctly skipping the face update for that direction [S1, unofficial].

## Subtopics

### [Static Mesh Caching](1138-static-mesh-caching.md)
Static Mesh Caching reduces draw calls and GPU data upload frequency by aggregating all voxel geometry into a single `Mesh` object. It differs from its parent [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by ignoring shared face optimization; instead of toggling visibility, it rebuilds the full geometry of every visible voxel into one buffer. It differs from [Dynamic Block Mesh Management](3574-dynamic-block-mesh-management.md) by maintaining a single global mesh rather than managing multiple independent mesh objects.

### [Dynamic Block Mesh Management](3574-dynamic-block-mesh-management.md)
Dynamic Block Mesh Management is a technique for maintaining a dynamic list of block indices organized by spatial coordinates, allowing for efficient addition and removal of individual block faces without rebuilding the entire mesh. It solves the performance bottleneck of static mesh regeneration in voxel environments where the geometry changes frequently. Unlike [Static Mesh Caching](1138-static-mesh-caching.md), which assumes immutable geometry, this method updates the index buffer in real-time by managing block-level data structures.

## Operating characteristics

## Background
Originating from the Shiranuit/VoxelEngine-0 project, this method is part of a broader effort in efficient voxel rendering [S1, unofficial].

## Sources
[S1] Shiranuit/VoxelEngine-0, https://github.com/Shiranuit/VoxelEngine-0, fetched 2026-10-27 (unofficial)
