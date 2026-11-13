# Greedy Quad Vertex Generation with AO and Anisotropy

Also called: Anisotropy flip logic, Triangulation and Anisotropy Correction

## Overview
This method converts a greedy mesh quad into four vertex indices packed into u32, applying ambient occlusion (AO) blending and correcting for interpolation anisotropy [S1, S2, unofficial]. It solves the problem where vertex-based AO values create visible seams or inconsistent shading gradients across a quad face due to the arbitrary direction of the splitting diagonal [S2, unofficial]. It differs from its parent [Binary Greedy Voxel Meshing with Ambient Occlusion](4465-binary-greedy-voxel-meshing-with-ambient-occlusion.md) by actively rotating the vertex order to ensure the diagonal connects vertices with higher combined occlusion, thereby smoothing the gradient [S1, S2, unofficial].

## How it works
**Inputs:**
- `GreedyQuad`: Struct containing `x` (u32), `y` (u32), `w` (u32), `h` (u32) [S1, unofficial].
- `ao`: u32 bitmask representing the solidity of surrounding voxels [S1, unofficial].
- `face_dir`: Face direction object with method `world_to_sample(axis, x, y, lod)` [S1, unofficial].
- `lod`: Level of detail object with method `jump_index()` [S1, unofficial].
- `normal_index`: Integer index for the vertex normal [S1, unofficial].
- `block_type`: Integer identifier for the block type [S1, unofficial].

**Outputs:**
- A sequence of four vertex indices (packed as u32) appended to the output vertex list [S1, unofficial].

**Data Structures:**
- `GreedyQuad`:
  - `x`: u32, horizontal position [S1, unofficial].
  - `y`: u32, vertical position [S1, unofficial].
  - `w`: u32, width [S1, unofficial].
  - `h`: u32, height [S1, unofficial].

**Steps:**

1. **Calculate AO Values for Corners:** Extract specific bits from the `ao` parameter to compute discrete shading tiers (0–3) for each corner [S1, unofficial].
   - $v1_{ao} = ((ao \gg 0) \& 1) + ((ao \gg 1) \& 1) + ((ao \gg 3) \& 1)$ [S1, unofficial].
   - $v2_{ao} = ((ao \gg 3) \& 1) + ((ao \gg 6) \& 1) + ((ao \gg 7) \& 1)$ [S1, unofficial].
   - $v3_{ao} = ((ao \gg 5) \& 1) + ((ao \gg 8) \& 1) + ((ao \gg 7) \& 1)$ [S1, unofficial].
   - $v4_{ao} = ((ao \gg 1) \& 1) + ((ao \gg 2) \& 1) + ((ao \gg 5) \& 1)$ [S1, unofficial].

2. **Generate Vertex Indices:** Create four vertex indices using `make_vertex_u32` [S1, unofficial]. The world position is calculated by mapping the quad's local coordinates to the world space using the face direction and LOD jump [S1, unofficial].
   - $v1 = \text{make\_vertex\_u32}(\text{face\_dir}.\text{world\_to\_sample}(\text{axis}, x, y, \text{lod}) \times \text{lod}.\text{jump\_index}(), v1_{ao}, \text{normal\_index}, \text{block\_type})$ [S1, unofficial].
   - $v2 = \text{make\_vertex\_u32}(\text{face\_dir}.\text{world\_to\_sample}(\text{axis}, x + w, y, \text{lod}) \times \text{lod}.\text{jump\_index}(), v2_{ao}, \text{normal\_index}, \text{block\_type})$ [S1, unofficial].
   - $v3 = \text{make\_vertex\_u32}(\text{face\_dir}.\text{world\_to\_sample}(\text{axis}, x + w, y + h, \text{lod}) \times \text{lod}.\text{jump\_index}(), v3_{ao}, \text{normal\_index}, \text{block\_type})$ [S1, unofficial].
   - $v4 = \text{make\_vertex\_u32}(\text{face\_dir}.\text{world\_to\_sample}(\text{axis}, x, y + h, \text{lod}) \times \text{lod}.\text{jump\_index}(), v4_{ao}, \text{normal\_index}, \text{block\_type})$ [S1, unofficial].

3. **Anisotropy Correction (Flip Logic):** Determine if the diagonal split should be flipped to ensure smooth shading [S2, unofficial]. Compare the AO values of the diagonal vertex pairs: calculate the sum of opposite corners ($AO_{00} + AO_{11}$) versus the other pair ($AO_{10} + AO_{01}$) [S2, unofficial]. Flip the quad's shared interior edge so that it connects the two vertices with the higher combined occlusion values [S2, unofficial]. Specifically, if the anisotropy condition $((v1_{ao} > 0) \oplus (v3_{ao} > 0))$ is true, rotate the vertex array right by one (move $v1$ to the end) [S1, unofficial].

4. **Append to Output:** Create a deque from $[v1, v2, v3, v4]$ and append the final sequence to the output vertices vector [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def make_vertex_u32(world_pos: int, ao_val: int, normal_idx: int, block_type: int) -> int:
    # Packs arguments into a single 32-bit unsigned integer. Packing layout not specified in sources [G].
    return (world_pos & 0xFFFF) | ((ao_val & 0xFF) << 16) | ((normal_idx & 0xFF) << 24) | ((block_type & 0xFF) << 31)

def generate_quad_vertices(quad_x: int, quad_y: int, quad_w: int, quad_h: int, ao: int, face_dir, lod, axis: int, normal_index: int, block_type: int) -> list[int]:
    # 1. Calculate AO values for corners [S1, unofficial]
    v1_ao = ((ao >> 0) & 1) + ((ao >> 1) & 1) + ((ao >> 3) & 1)
    v2_ao = ((ao >> 3) & 1) + ((ao >> 6) & 1) + ((ao >> 7) & 1)
    v3_ao = ((ao >> 5) & 1) + ((ao >> 8) & 1) + ((ao >> 7) & 1)
    v4_ao = ((ao >> 1) & 1) + ((ao >> 2) & 1) + ((ao >> 5) & 1)

    # 2. Generate vertex indices [S1, unofficial]
    pos1 = face_dir.world_to_sample(axis, quad_x, quad_y, lod) * lod.jump_index()
    pos2 = face_dir.world_to_sample(axis, quad_x + quad_w, quad_y, lod) * lod.jump_index()
    pos3 = face_dir.world_to_sample(axis, quad_x + quad_w, quad_y + quad_h, lod) * lod.jump_index()
    pos4 = face_dir.world_to_sample(axis, quad_x, quad_y + quad_h, lod) * lod.jump_index()

    v1 = make_vertex_u32(pos1, v1_ao, normal_index, block_type)
    v2 = make_vertex_u32(pos2, v2_ao, normal_index, block_type)
    v3 = make_vertex_u32(pos3, v3_ao, normal_index, block_type)
    v4 = make_vertex_u32(pos4, v4_ao, normal_index, block_type)

    # 3. Anisotropy Correction [S1, S2, unofficial]
    # Compare diagonal AO sums [S2, unofficial]
    diag_sum_1 = v1_ao + v3_ao
    diag_sum_2 = v2_ao + v4_ao

    # Apply rotation if anisotropy condition is met [S1, unofficial]
    if (v1_ao > 0) ^ (v3_ao > 0):
        vertices = [v2, v3, v4, v1]
    else:
        vertices = [v1, v2, v3, v4]

    return vertices
```

**Edge Cases and Pitfalls:**
- If vertices with identical AO values are placed across the wrong diagonal, the quad renders with a visible visual seam or inconsistent shading gradient [S2, unofficial].
- The XOR condition $((v1_{ao} > 0) \oplus (v3_{ao} > 0))$ handles cases where one diagonal has occlusion and the other does not, but the implementation uses this rotation as the primary correction heuristic rather than the theoretical optimal diagonal flip based on AO sums [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, S2, unofficial].
- **Memory Complexity:** Not given in the sources [S1, S2, unofficial].
- **Strengths:** Reduces visual artifacts (seams) in AO shading without adding geometry [S2, unofficial]. Integrates seamlessly with greedy meshing by preserving quad topology [S1, unofficial].
- **Limitations:** The correction heuristic (XOR rotation) may not perfectly match the theoretical optimal diagonal flip based on AO sums [S1, S2, unofficial]. Relies on discrete AO tiers, which can lead to banding if not smoothed [S2, unofficial].

## Background
This method is part of the binary greedy mesher implementation [S1, unofficial]. It addresses the conflict between greedy meshing (which merges faces) and ambient occlusion (which varies per voxel) by baking AO into vertex data and correcting interpolation artifacts [S2, unofficial]. The anisotropy flip logic is a specific optimization to ensure smooth shading across merged quads [S2, unofficial].

## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S2] Baking Dynamic Ambient Occlusion in Procedural Voxels, https://salivity.github.io/game-development/article/baking-dynamic-ambient-occlusion-in-procedural-voxels, fetched 2026-10-29 (unofficial)
