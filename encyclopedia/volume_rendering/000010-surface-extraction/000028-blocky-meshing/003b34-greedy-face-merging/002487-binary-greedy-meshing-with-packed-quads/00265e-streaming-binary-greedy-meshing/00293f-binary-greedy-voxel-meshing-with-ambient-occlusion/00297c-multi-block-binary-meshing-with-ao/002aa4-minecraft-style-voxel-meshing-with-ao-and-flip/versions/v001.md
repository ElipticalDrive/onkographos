# Minecraft-style Voxel Meshing with AO and Flip

Also called: Greedy Mesh Generation, Minecraft-style Voxel Meshing, Voxel Face Mesh Generation with AO and Flip

## Overview
This method generates a mesh for a voxel segment by merging adjacent blocks of the same type into larger rectangular faces, reducing draw calls and vertex count [S2, unofficial]. It extends binary greedy meshing by incorporating ambient occlusion (AO) values into the vertex data and using a "flip" bit to optimize triangle winding order based on AO distribution [S1, unofficial]. It builds on [Multi-Block Binary Meshing with AO](4534-multi-block-binary-meshing-with-ao.md), which handles the iteration over block types and axes, by adding the specific logic for AO-based flip determination and vertex packing for the final geometry generation.

## How it works
**Inputs:** A voxel segment (typically 16x16x16 blocks), a visibility mask, and block type data.
**Outputs:** A list of vertices (position, color, packed data) and an index buffer for each face direction.

**Data Structures:**
1.  `mask`: Array of 4096 integers (for a 16x16x16 segment). `mask[i]` stores the block type ID. If the block is null or not visible, it stores `BlockType.AIR`.
2.  `done`: Array of 4096 booleans. `done[i]` is true if the block at index `i` has already been included in a merged face.
3.  `v0, v1, v2, v3`: Vertex structures containing position, color, and packed data.
4.  `packed_data`: A packed integer or struct containing world coordinates, `voxel_id`, `face_id`, `ao` value, and `flip_id`.

**Parameters:**
*   `segment_size`: 16 (standard Minecraft chunk segment size).
*   `face_id`: Integer distinguishing the face direction (e.g., 4 for TOP, 5 for FRONT).
*   `AIR`: Constant representing empty space.

**Steps:**

1.  **Initialize Mask:**
    Iterate `i` from 0 to 4095.
    *   If `segment.getBlock(i)` is null or `!segment.getBlock(i).isVisible()`, set `mask[i] = AIR`.
    *   Else, set `mask[i] = segment.getBlock(i).getType()`.
    *   Initialize `done[i] = false` for all `i`.

2.  **Iterate Face Directions:**
    Loop through the 6 face directions `i` (0=TOP, 1=BOT, 2=LEFT, 3=RIGHT, 4=FRONT, 5=BACK).
    *   Skip if the face is not visible to the camera (culling).
    *   Pre-check: If all blocks in the segment have `mask[j] == AIR` or the face is not visible for any block, skip this face direction.

3.  **Scan for Merged Faces:**
    Iterate `y` from 0 to 15, `z` from 0 to 15, `x` from 0 to 15.
    *   Calculate linear index: `sum = x + (z << 4) + (y << 8)`.
    *   If `done[sum]` is true, continue.
    *   Let `type = mask[sum]`. If `type == AIR`, set `done[sum] = true` and continue.
    *   If the block at `sum` does not have the current face `i` visible, set `done[sum] = true` and continue.

4.  **Determine Dimensions (Greedy Expansion):**
    Find the width `w` and height `h` of the largest rectangle of blocks with `type` that are not `done`.
    
    *   **For TOP/BOT faces (x-z plane):**
        *   Find `w`: Iterate `w` from 0 while `w < 15 - x` AND `mask[sum + 1 + w] == type` AND `!done[sum + 1 + w]`. (Note: Check boundary first to avoid out-of-bounds access).
        *   Find `h`: Iterate `h` from 0 while `h < 15 - z`.
            *   For each `h`, check if all blocks in the row `sum + (h << 4) + k` for `k` from 0 to `w` match `type` and are not `done`.
            *   If any block fails, break.
        *   Mark all cells in the rectangle `(x, z)` to `(x+w, z+h)` as `done`.

    *   **For LEFT/RIGHT faces (z-y plane):**
        *   Find `w` (along z-axis): Iterate `w` from 0 while `w < 15 - z` AND `mask[sum + (w << 4) + 1] == type` AND `!done[sum + (w << 4) + 1]`.
        *   Find `h` (along y-axis): Iterate `h` from 0 while `h < 15 - y`.
            *   For each `h`, check if all blocks in the column `sum + (h << 8) + (k << 4)` for `k` from 0 to `w` match `type` and are not `done`.
            *   If any block fails, break.
        *   Mark all cells in the rectangle as `done`.

    *   **For FRONT/BACK faces:**
        *   The source notes indicate these are handled similarly or skipped in the specific snippet, but the general greedy logic applies to the x-y plane.

5.  **Calculate Ambient Occlusion and Flip ID:**
    For the four corners of the generated rectangle, calculate AO values using `get_ao`. Let the corners be indexed 0, 1, 2, 3.
    *   `ao[0]`, `ao[1]`, `ao[2]`, `ao[3]` are the AO values for the corners.
    *   Determine `flip_id`: `flip_id = (ao[1] + ao[3]) > (ao[0] + ao[2])`.
    *   This boolean determines the winding order of the triangles to optimize for AO shading continuity.

6.  **Pack Vertex Data:**
    For each of the 4 corners of the rectangle, create a vertex `v` using `pack_data`.
    *   Arguments: World coordinates (e.g., `bX`, `bY`, `bZ`), `voxel_id`, `face_id`, `ao` value, and `flip_id`.
    *   Store these in `v0, v1, v2, v3`.

7.  **Generate Geometry:**
    Call `addVertices(type.getColor(), i, bX, bY, bZ, w, h)`.
    
    *   **Vertex Positions:**
        *   **TOP:**
            *   `v0`: `(x - 0.5, y + 0.5, z - 0.5)`
            *   `v1`: `(x - 0.5, y + 0.5, z + 0.5 + h)`
            *   `v2`: `(x + 0.5 + w, y + 0.5, z + 0.5 + h)`
            *   `v3`: `(x + 0.5 + w, y + 0.5, z - 0.5)`
        *   **BOT:**
            *   `v0`: `(x - 0.5, y - 0.5, z - 0.5)`
            *   `v1`: `(x + 0.5 + w, y - 0.5, z - 0.5)`
            *   `v2`: `(x + 0.5 + w, y - 0.5, z + 0.5 + h)`
            *   `v3`: `(x - 0.5, y - 0.5, z + 0.5 + h)`
        *   **LEFT:**
            *   `v0`: `(x - 0.5, y + 0.5 + h, z + 0.5 + w)`
            *   `v1`: `(x - 0.5, y + 0.5 + h, z - 0.5)`
            *   `v2`: `(x - 0.5, y - 0.5, z - 0.5)`
            *   `v3`: `(x - 0.5, y - 0.5, z + 0.5 + w)`
        *   **RIGHT/FRONT/BACK:** Positions follow similar logic relative to the face normal and dimensions.

    *   **Indices:**
        *   If `flip_id` is true, add indices: `(v3, v1, v0)` and `(v3, v2, v1)`.
        *   If `flip_id` is false, add indices: `(v0, v2, v1)` and `(v0, v3, v2)`.
        *   (Note: The specific vertex order in the indices depends on the corner assignment in `pack_data` and the face orientation. The source notes specify `v3, v1, v0, v3, v2, v1` for flip=true and `v0, v2, v1, v0, v3, v2` for flip=false).

**Pitfalls:**
*   **Boundary Check:** When finding width `w`, the condition `mask[sum + w] == type` checks the block at `w=0` which is already known to be `type`. The loop should check `mask[sum + 1 + w]` to find the *next* block, and must check `w < 15 - x` *before* accessing the array to prevent `ArrayIndexOutOfBoundsException` [S2, unofficial].
*   **AO Calculation:** `get_ao` must correctly sample neighboring blocks to determine the darkness of each corner.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of blocks in the segment (4096 for a 16x16x16 chunk). Each block is visited a constant number of times during the greedy expansion.
*   **Memory Complexity:** $O(N)$ for the `mask` and `done` arrays.
*   **Performance:** Significantly reduces vertex count and draw calls compared to per-block meshing. The greedy merging is efficient for large uniform areas.
*   **Strengths:** Simple to implement, effective for voxel games with blocky geometry, integrates AO for shading.
*   **Limitations:** Only merges axis-aligned rectangles, so diagonal or complex shapes are not optimized. The "flip" optimization is specific to the AO winding order and may not apply to all shading models.
*   **When to use:** Voxel-based games with static or infrequently changing chunks.
*   **When not to use:** Dynamic meshes with frequent updates, or non-voxel geometry.

## Background
The method originates from community implementations for voxel-based games like Minecraft, where reducing draw calls is critical for performance. The specific "flip" logic based on AO sums is documented in GitHub repositories and game development forums [S1, S2, unofficial]. The greedy meshing approach itself is a well-known optimization in voxel rendering, first popularized by the need to render large Minecraft worlds efficiently.

## Sources
[S1] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S2] Greedy Mesh generation for a voxel-based game, https://gamedev.stackexchange.com/questions/106168, fetched 2026-10-28 (unofficial)
