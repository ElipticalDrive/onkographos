# Multi-Block Binary Meshing with AO
## Overview
Multi-Block Binary Meshing with AO generates mesh vertices for a specific face direction within a voxel chunk by iterating over all block types and axes, packing ambient occlusion (AO) values into a vertex index (`p_index`) [S1, unofficial]. It builds on [Binary Greedy Voxel Meshing with Ambient Occlusion](4465-binary-greedy-voxel-meshing-with-ambient-occlusion.md) [G]. Unlike its sibling [Binary Greedy Voxel Meshing without Ambient Occlusion](4536-binary-greedy-voxel-meshing-without-ambient-occlusion.md), which treats all non-air blocks as opaque and outputs vertices with `ao=0`, this method preserves per-vertex AO values derived from surrounding voxel solidity [S1, unofficial].
## How it works
**Inputs:**
- `lod`: Voxel chunk data structure containing binary voxel states.
- `face_dir`: Direction of the face being extracted.
- `MESHABLE_BLOCK_TYPES`: List of block types requiring meshing.

**Outputs:**
- `vertices`: Vector of mesh vertices containing position, AO value, and block type.

**Data Structures:**
- `p_index`: 32-bit integer packing vertex metadata. Lower 9 bits store `ao` (`p_index & 0b111111111`), and bits 9 and above store `block_type` (`p_index >> 9`) [S1, unofficial].
- `x_data`: HashMap or array mapping `p_index` to a `[u32; 32]` array representing the binary plane data for that specific block type and AO combination. If a `p_index` is not present, it is initialized with `[0u32; 32]` [S1, unofficial].

**Steps:**
1. Initialize an empty `vertices` vector.
2. Iterate over each axis from 0 to `size` (dimension of the chunk) [S1, unofficial].
3. Iterate over each `block_type` in `MESHABLE_BLOCK_TYPES` [S1, unofficial].
4. For each axis and block type, initialize `x_data` as a HashMap mapping `p_index` to `[u32; 32]` [S1, unofficial].
5. For each `p_index` and `data` in `x_data`:
   a. Extract `ao` from `p_index` using bitwise AND: `ao = p_index & 0b111111111` [S1, unofficial].
   b. Extract `block_type` from `p_index` using bitwise shift: `block_type = p_index >> 9` [S1, unofficial].
   c. Call `greedy_mesh_binary_plane(data, lod.size())` to get a list of `GreedyQuad` objects [S1, unofficial].
   d. For each `quad` in the result:
      i. Call `append_vertices(vertices, face_dir, axis, lod, ao, block_type)` to generate and add the mesh vertices [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_block_binary_meshing_with_ao(lod, face_dir, meshable_block_types):
    vertices = []
    lod_size = lod.size()
    for axis in range(3):
        for block_type in meshable_block_types:
            x_data = {}
            # x_data is populated from voxel traversal; source iterates over its items
            for p_index, data in x_data.items():
                ao = p_index & 0b111111111
                current_block_type = p_index >> 9
                quads = greedy_mesh_binary_plane(data, lod_size)
                for quad in quads:
                    append_vertices(vertices, face_dir, axis, lod, ao, current_block_type)
    return vertices

def greedy_mesh_binary_plane(data, lod_size):
    quads = []
    for row in range(len(data)):
        y = 0
        while y < lod_size:
            shifted = data[row] >> y
            if shifted == 0:
                y = lod_size
                break
            tz = (shifted & -shifted).bit_length() - 1
            y += tz
            if y >= lod_size:
                break
            ones = shifted >> tz
            h = 0
            while ones & (1 << h):
                h += 1
            quads.append((row, y, h))
            y += h
    return quads

def append_vertices(vertices, face_dir, axis, lod, ao, block_type):
    # Generate 4 corner vertices for the quad in the current plane [G]
    v0, v1, v2, v3, v4 = generate_quad_vertices(axis, lod)
    v1ao, v3ao = ao, ao
    new_vertices = [v1, v2, v3, v4]
    if face_dir.reverse_order():
        new_vertices = [new_vertices[0]] + new_vertices[1:][::-1]
    if (v1ao > 0) ^ (v3ao > 0):
        new_vertices = new_vertices[1:] + [new_vertices[0]]
    vertices.extend(new_vertices)

def generate_quad_vertices(axis, lod):
    # Returns 4 corner vertices for the current quad based on axis and lod dimensions [G]
    return [(0,0,0), (1,0,0), (1,1,0), (0,1,0), (0,0,0)]
```

**Parameters:**
- `lod.size()`: The dimension of the voxel chunk (e.g., 16, 32) [S1, unofficial].
- `MESHABLE_BLOCK_TYPES`: List of block types to process [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundaries:** The loop `y += (data[row] >> y).trailing_zeros()` must handle cases where `data[row]` is 0 to avoid infinite loops or undefined behavior [S1, unofficial].
- **Degenerate Input:** Empty chunks or chunks with no meshable blocks should return an empty vertex list [G].
- **Precision:** AO values are packed into 9 bits. Ensure AO calculations fit within this range [S1, unofficial].
## Subtopics

### [Binary Greedy Voxel Meshing without Ambient Occlusion](4536-binary-greedy-voxel-meshing-without-ambient-occlusion.md)
This method generates mesh vertices for a specific face direction in a voxel chunk without computing ambient occlusion (AO) values. It iterates over all block types and axes, using a fixed-size array for spatial data to drive the greedy meshing process. Unlike its sibling [Binary Greedy Voxel Meshing with Ambient Occlusion](4536-binary-greedy-voxel-meshing-with-ambient-occlusion.md), which packs AO data into the vertex index, this variant treats all non-air blocks as opaque for culling purposes and outputs vertices with `ao=0` and `block_type=0`. It serves as the foundational geometry generation step before AO or transparency extensions are applied.

### Minecraft-style Voxel Meshing with AO and Flip
This variant calculates AO values for the four corners of each face using `get_ao` [S2, unofficial]. It determines a `flip_id` boolean: `flip_id = ao[1] + ao[3] > ao[0] + ao[2]` [S2, unofficial]. Vertex data for the four corners is packed into `v0, v1, v2, v3` using `pack_data`, which includes world coordinates, `voxel_id`, a face identifier (e.g., 4 for top, 5 for front), AO value, and `flip_id` [S2, unofficial]. If `flip_id` is true, vertices are added to the index buffer in the order `v3, v1, v0, v3, v2, v1`; if false, they are added in the order `v0, v2, v1, v0, v3, v2` [S2, unofficial]. Vertex data is packed into a buffer via `add_data` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels and block types [G].
- **Memory Complexity:** Proportional to the number of unique `p_index` values, as each requires a `[u32; 32]` array in `x_data` [G].
- **Performance:** Efficient for sparse voxel worlds due to binary plane representation and greedy meshing [G].
- **Strengths:** Supports multiple block types, preserves AO information, and reduces overdraw via flip optimization [G].
- **Limitations:** Complex packing of AO and block type into `p_index` may limit the range of AO values or block types [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
## Background
Multi-Block Binary Meshing with AO is an extension of binary greedy meshing techniques used in voxel-based games [S1, unofficial]. It was developed to support multiple block types while maintaining the performance benefits of binary plane representation and AO [S1, unofficial]. The method is commonly used in voxel engines that require detailed surface rendering with ambient occlusion [S1, unofficial].
## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
