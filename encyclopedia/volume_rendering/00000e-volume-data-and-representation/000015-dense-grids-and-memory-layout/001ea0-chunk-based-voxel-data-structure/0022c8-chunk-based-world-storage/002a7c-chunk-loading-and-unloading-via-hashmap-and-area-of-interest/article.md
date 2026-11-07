# Chunk Loading and Unloading via Hashmap and Area of Interest

Also called: AoI-based Chunk Management, Hashmap Chunk Culling

## Overview
This method manages the lifecycle of voxel chunks by maintaining a hash map of active chunks and culling those outside a defined Area of Interest (AoI) centered on the camera [S1, unofficial]. It solves the problem of memory exhaustion and rendering overhead in large voxel worlds by ensuring only visible or nearby geometry is resident in memory [S1, unofficial]. It differs from static world storage by dynamically allocating and deallocating chunk objects based on camera position [S1, unofficial].

## How it works
**Inputs:** Camera position $(C_x, C_y, C_z)$, render distance $R$, and a hash map $H$ mapping chunk coordinates to chunk objects [S1, unofficial].
**Outputs:** A set of chunks to render and an updated hash map $H$ [S1, unofficial].

**Data Structures:**
*   **Chunk Class:** Must disable copy and move constructors and assignment operators to prevent accidental copying and aid in debugging memory management [S1, unofficial].
*   **Hash Map $H$:** Maps 3D integer coordinates $(cx, cy, cz)$ to `Chunk` pointers. The hash function must handle tuples, as standard libraries may not provide a built-in hash for 3D tuples [S1, unofficial].

**Steps:**
1.  **Define AoI:** Calculate the bounding cube of the Area of Interest as:
    $$ [C_x - R, C_x + R] \times [C_y - R, C_y + R] \times [C_z - R, C_z + R] $$
    where $R$ is the render distance [S1, unofficial].
2.  **Iterate and Load:** Iterate through all integer chunk coordinates $(cx, cy, cz)$ within the AoI bounds [S1, unofficial].
    *   If $(cx, cy, cz)$ exists in $H$, fetch the chunk object [S1, unofficial].
    *   If $(cx, cy, cz)$ does not exist in $H$, create a new chunk object, insert it into $H$, and fetch it [S1, unofficial].
3.  **Render:** Render all fetched chunks [S1, unofficial].
4.  **Unload:** Identify all chunks in $H$ whose coordinates lie outside the AoI. Delete these chunk objects and remove them from $H$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_chunks(camera_pos, render_distance, chunk_hashmap):
    # 1. Define AoI bounds (floored/ceiled to chunk coordinates)
    min_cx = int(floor(camera_pos.x - render_distance))
    max_cx = int(ceil(camera_pos.x + render_distance))
    min_cy = int(floor(camera_pos.y - render_distance))
    max_cy = int(ceil(camera_pos.y + render_distance))
    min_cz = int(floor(camera_pos.z - render_distance))
    max_cz = int(ceil(camera_pos.z + render_distance))

    # 2. Load chunks inside AoI
    chunks_to_render = []
    cx = min_cx
    while cx <= max_cx:
        cy = min_cy
        while cy <= max_cy:
            cz = min_cz
            while cz <= max_cz:
                key = (cx, cy, cz)
                if key in chunk_hashmap:
                    chunks_to_render.append(chunk_hashmap[key])
                else:
                    # Allocate new chunk object [G]
                    new_chunk = Chunk(key)
                    chunk_hashmap[key] = new_chunk
                    chunks_to_render.append(new_chunk)
                cz += 1
            cy += 1
        cx += 1

    # 3. Unload chunks outside AoI
    keys_to_remove = []
    for key, chunk in chunk_hashmap.items():
        if not (min_cx <= key[0] <= max_cx and 
                min_cy <= key[1] <= max_cy and 
                min_cz <= key[2] <= max_cz):
            keys_to_remove.append(key)
    
    for key in keys_to_remove:
        # Delete chunk object [G]
        del chunk_hashmap[key]

    return chunks_to_render
```

**Parameters:**
*   **Render Distance ($R$):** The radius defining the AoI [S1, unofficial].

**Edge Cases:**
*   **Memory Management:** Disabling copy/move semantics is critical to avoid double-free errors or hidden copies during debugging [S1, unofficial].
*   **Hashing:** Standard libraries may lack tuple hashing; a custom hash function for 3D coordinates is required [S1, unofficial].

## Subtopics

### Underground Sub-chunk Culling
An optimization where sub-chunks (smaller subdivisions within a chunk) are culled when the camera is underground. This results in up to 70x fewer sub-chunks being drawn and loaded compared to non-underground scenarios [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(V)$ where $V$ is the number of voxels in the AoI, assuming constant-time hash map lookups and chunk creation [G]. Unloading is $O(C)$ where $C$ is the number of active chunks [G].
**Memory Complexity:** Proportional to the number of chunks within the AoI [G].
**Performance:** On an Apple M4, rendering 277 million voxels (4,225 chunks) at radius 32 took 8.2 ms [S2, unofficial]. Underground optimization can reduce loaded sub-chunks by up to 70x [S2, unofficial].

## Background
This method is part of the "Mutiny" voxel engine implementation, focusing on chunk lifecycle management via hash maps and AoI culling [S1, unofficial]. It builds on the chunk-based world storage model [2858-chunk-based-world-storage.md] [S1, unofficial].

## Sources
[S1] kotychok/mutiny, https://github.com/kotychok/mutiny, fetched 2026-10-27 (unofficial)
[S2] DanielWLiu07/voxel-engine, https://github.com/DanielWLiu07/voxel-engine, fetched 2026-10-26 (unofficial)
