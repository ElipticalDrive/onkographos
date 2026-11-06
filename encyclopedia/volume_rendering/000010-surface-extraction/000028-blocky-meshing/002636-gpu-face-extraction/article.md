# GPU Face Extraction

Also called: GPU Face Mesher, Persistent Face Mesher

## Overview
GPU Face Extraction is a compute-shader-based algorithm that extracts visible surface faces from a voxel subchunk and writes packed quad descriptors to a persistent buffer, avoiding per-frame mesh generation by keeping descriptors resident until the subchunk becomes dirty [S1, unofficial]. It differs from CPU-based meshing by offloading the visibility culling and quad packing to the GPU, and from greedy meshing by outputting individual exposed faces rather than merged quads, relying on the GPU's parallel atomic operations to handle concurrent writes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A padded 18×18×18 voxel cache containing block IDs for a 16×16×16 subchunk, including a 1-voxel border for neighbor lookups [S1, unofficial].
- Output: A buffer of `PackedQuad` structures representing visible faces, and metadata in `SubchunkMeta` storing the total vertex count [S1, unofficial].

**Data Structures:**
1.  **Voxel Buffer:** A 1D array of `u32` block IDs, size $18^3 = 5832$ elements. Index mapping:
    $$ \text{voxel\_index}(x, y, z) = x \cdot 324 + y \cdot 18 + z $$
    where $x, y, z \in [0, 17]$ [S1, unofficial].
2.  **PackedQuad:** A 16-byte struct (4 `u32` fields) packed as follows [S1, unofficial]:
    -   `origin_and_face` (u32): Bits 0-1: $x \pmod 2$; Bits 2-7: $y \pmod 2$; Bits 8-13: $z \pmod 2$; Bits 14-15: face index (0-5).
        -   Formula: $\text{ox} = (x-1)*2 + (\text{face}==1?2:0)$, $\text{oy} = (y-1)*2 + (\text{face}==3?2:0)$, $\text{oz} = (z-1)*2 + (\text{face}==5?2:0)$.
        -   Packed: $\text{ox} \mid (\text{oy} \ll 6) \mid (\text{oz} \ll 12) \mid (\text{face} \ll 18)$.
    -   `size_material_ao` (u32): Bits 0-4: AO (0-3); Bits 5: 1; Bits 10-14: material index (0-17); Bits 18-31: 0xFF.
        -   Packed: $1 \mid (1 \ll 5) \mid (\text{tex}(\text{block}, \text{face}) \ll 10) \mid (0\text{xff} \ll 18)$.
    -   `color_flags` (u32): Derived from `color(face)` [S1, unofficial].
    -   `reserved` (u32): Padding [S1, unofficial].
3.  **SubchunkMeta:** Struct `{ world_origin: vec4<i32>, draw_data: vec4<u32> }`. `draw_data.x` stores `count * 6` (total vertices) [S1, unofficial].
4.  **FaceExtractUniforms:** Struct `{ output_quad_offset: u32, quad_capacity: u32, subchunk_slot: u32, _padding: u32 }` [S1, unofficial].
5.  **Face Counter:** A single `u32` atomic counter for quad output indexing [S1, unofficial].

**Parameters:**
-   `PADDED_SIDE`: 18 [S1, unofficial].
-   `SUBCHUNK_SIZE`: 16 [S1, unofficial].
-   Workgroup size: 64 threads per workgroup, 64 workgroups dispatched for `extract_faces` (4096 voxels) [S1, unofficial].
-   `finalize_faces`: 1 workgroup (1 thread) [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    -   Upload voxel IDs from the snapshot cache to `voxel_buffer` [S1, unofficial].
    -   Reset `face_counter` to 0 [S1, unofficial].
    -   Populate `FaceExtractUniforms` with buffer offsets and capacities [S1, unofficial].
    -   Bind buffers: `voxel_buffer` (slot 0), `quad_buffer` (slot 1), `subchunk_meta_buffer` (slot 2), `face_counter` (slot 3), `uniforms` (slot 4) [S1, unofficial].
2.  **Face Extraction (`extract_faces`):**
    -   Each thread processes one voxel $(x,y,z)$ in the 16×16×16 interior.
    -   For each of the 6 faces, check visibility:
        -   Get neighbor block ID using the padded index.
        -   Apply occlusion rules:
            -   Water (block 5) is visible only if neighbor is water [S1, unofficial].
            -   Transparent blocks (air, leaves, ice, cactus, dead bush) are visible if neighbor is transparent or air [S1, unofficial].
            -   Leaves (block 7) are visible only if neighbor is leaves [S1, unofficial].
        -   If visible, compute `PackedQuad` fields.
        -   Atomically increment `face_counter`. If new value $< quad\_capacity$, write `PackedQuad` to `quad_buffer` at `output_quad_offset + new_value`. Else, discard (buffer overflow guard) [S1, unofficial].
3.  **Finalization (`finalize_faces`):**
    -   Read final `face_counter` value.
    -   Update `SubchunkMeta.draw_data.x` with `face_counter * 6` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader: extract_faces
# Thread ID maps to voxel (x, y, z) in 0..15 range
x = thread_id % 16
y = (thread_id // 16) % 16
z = thread_id // 256

block_id = voxel_buffer[voxel_index(x + 1, y + 1, z + 1)] # Offset by 1 due to padding

for face in 0..5:
    # Determine neighbor coordinates based on face normal
    # Face 0: -X, Face 1: +X, Face 2: -Y, Face 3: +Y, Face 4: -Z, Face 5: +Z
    nx, ny, nz = get_neighbor_coords(x, y, z, face)
    
    # Clamp to padded bounds for lookup (padding handles boundary)
    neighbor_id = voxel_buffer[voxel_index(nx, ny, nz)]
    
    if is_visible(block_id, neighbor_id):
        # Calculate quad corner coordinates
        ox = (x - 1) * 2 + (2 if face == 1 else 0)
        oy = (y - 1) * 2 + (2 if face == 3 else 0)
        oz = (z - 1) * 2 + (2 if face == 5 else 0)
        
        # Pack origin_and_face
        origin_and_face = ox | (oy << 6) | (oz << 12) | (face << 18)
        
        # Pack size_material_ao
        tex_id = get_texture_index(block_id, face)
        size_material_ao = 1 | (1 << 5) | (tex_id << 10) | (0xFF << 18)
        
        color_flags = get_color_flags(face)
        
        # Atomic append
        idx = atomic_add(face_counter, 1)
        if idx < quad_capacity:
            quad_buffer[output_quad_offset + idx] = PackedQuad(origin_and_face, size_material_ao, color_flags, 0)

# Compute Shader: finalize_faces
total_faces = load(face_counter)
subchunk_meta.draw_data.x = total_faces * 6
```

**Edge Cases and Pitfalls:**
-   **Buffer Overflow:** If `atomicAdd` returns a value $\ge$ `quad_capacity`, the quad is not emitted to prevent out-of-bounds writes [S1, unofficial].
-   **Unsupported Blocks:** Water, DeadBush, and WoodStairs are excluded from GPU meshing and fall back to CPU processing [S1, unofficial].
-   **Precision:** Coordinates are packed into limited bit fields; ensure `ox`, `oy`, `oz` fit within the allocated bits (0-63 range for coordinates in the packed struct) [S1, unofficial].

## Subtopics

### [Validator Reference Face Reconstruction](3794-validator-reference-face-reconstruction.md)
A deterministic CPU-based algorithm used to generate a canonical list of surface faces from a voxel chunk for testing purposes. It serves as the ground truth against which GPU Face Extraction is validated. Unlike the GPU method, it iterates every voxel and neighbor, reconstructing individual face tokens without packing or atomic optimization, ensuring a complete representation of mesh topology.

### Face Orientation Consistency
A post-processing step that flips faces so that orientation among touching faces is consistent across connected components. It uses BFS/DFS to traverse connected faces, swapping vertices of adjacent faces if their shared vertices are not adjacent in index order, and finally flipping components based on the sum of normals at the maximum X-coordinate vertex [S3, unofficial].

### Chunk Face Extraction
Extracts the 6 boundary faces of a chunk into `ChunkFace` structures, mapping block IDs to visibility types. It iterates over all 6 directions, calculates block positions on each face, and maps block IDs to `VoxelBlock` types using a visibility checker, storing results in a flat array of size `CHUNK_SIZE^2` [S4, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the subchunk (4096), as each voxel is processed by one thread [S1, unofficial].
-   **Memory Complexity:** $O(F)$ for output faces, where $F$ is the number of visible faces. The padded voxel buffer is fixed at $18^3$ `u32`s [S1, unofficial].
-   **Performance:** Real-time mesh generation on GPU using wgpu and WGSL shaders, suitable for dynamic voxel worlds [S1, unofficial].
-   **Strengths:** Parallel visibility culling, persistent buffers reduce CPU-GPU transfer overhead for unchanged subchunks [S1, unofficial].
-   **Limitations:** Does not support all block types (e.g., Water, Stairs), requiring CPU fallback. Outputs individual faces, not merged quads, which may result in higher primitive counts than greedy meshing [S1, unofficial].
-   **When to Use:** Real-time voxel rendering where subchunks change infrequently and GPU compute is available [S1, unofficial].
-   **When Not to Use:** When memory bandwidth is limited and merged quads are preferred, or when unsupported block types dominate [S1, unofficial].

## Background
GPU Face Extraction is implemented in the Minerust project, a voxel game engine using Rust and wgpu. It leverages WGSL compute shaders for parallel face extraction, building on the principles of blocky meshing and face culling [S1, unofficial]. The method is part of a broader effort to optimize voxel rendering by offloading mesh generation to the GPU [S1, unofficial].

## Sources
[S1] B4rtekk1/Minerust, https://github.com/B4rtekk1/Minerust, fetched 2026-10-27 (unofficial)
[S2] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S3] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
[S4] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
