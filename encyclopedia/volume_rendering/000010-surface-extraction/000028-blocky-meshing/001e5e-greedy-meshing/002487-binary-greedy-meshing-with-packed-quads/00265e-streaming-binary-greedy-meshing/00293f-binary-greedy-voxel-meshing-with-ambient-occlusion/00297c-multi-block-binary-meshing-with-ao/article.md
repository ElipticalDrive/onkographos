# Multi-Block Binary Meshing with AO


## Overview
Multi-Block Binary Meshing with AO is a surface extraction method that generates mesh vertices for a specific face direction within a voxel chunk by iterating over all block types and axes, handling ambient occlusion (AO) by packing it into the vertex index (`p_index`) [S1, unofficial]. It builds on [Binary Greedy Voxel Meshing with Ambient Occlusion](4465-binary-greedy-voxel-meshing-with-ambient-occlusion.md) by extending the greedy meshing process to support multiple block types simultaneously, where each type is processed independently but shares the same binary plane data structure. Unlike its sibling [Binary Greedy Voxel Meshing without Ambient Occlusion](4536-binary-greedy-voxel-meshing-without-ambient-occlusion.md), which treats all non-air blocks as opaque and outputs vertices with `ao=0`, this method preserves per-vertex AO values derived from the surrounding voxel solidity [S1, unofficial].

## How it works
**Inputs:**
- `lod`: A voxel chunk data structure containing binary voxel states.
- `face_dir`: The direction of the face being extracted (e.g., +X, -Y).
- `MESHABLE_BLOCK_TYPES`: A list of block types that require meshing.

**Outputs:**
- `vertices`: A vector of mesh vertices, each containing position, AO value, and block type.

**Data Structures:**
- `p_index`: A 32-bit integer used to pack vertex metadata.
  - Bits 0-8 (9 bits): `block_type` (shifted right by 9: `p_index >> 9`).
  - Bits 9-31 (23 bits): Unused or reserved in the extraction logic shown, but AO is extracted from the lower bits.
  - Bits 0-8 (specifically lower 9 bits in some contexts, but source specifies `& 0b111111111` which is 9 bits): `ao` value. Note: Source [S1, unofficial] extracts AO using `p_index & 0b111111111`.
- `x_data`: A HashMap or array mapping `p_index` to a `[u32; 32]` array representing the binary plane data for that specific block type and AO combination. If a `p_index` is not present, it is initialized with `[0u32; 32]`.

**Steps:**
1. Initialize an empty `vertices` vector.
2. Iterate over each axis from 0 to `size` (dimension of the chunk).
3. Iterate over each `block_type` in `MESHABLE_BLOCK_TYPES`.
4. For each axis and block type, initialize `x_data` as a HashMap mapping `p_index` to `[u32; 32]`.
5. For each `p_index` and `data` in `x_data`:
   a. Extract `ao` from `p_index` using bitwise AND: `ao = p_index & 0b111111111`.
   b. Extract `block_type` from `p_index` using bitwise shift: `block_type = p_index >> 9`.
   c. Call `greedy_mesh_binary_plane(data, lod.size())` to get a list of `GreedyQuad` objects.
   d. For each `quad` in the result:
      i. Call `append_vertices(vertices, face_dir, axis, lod, ao, block_type)` to generate and add the mesh vertices.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_block_binary_meshing_with_ao(lod, face_dir, meshable_block_types):
    vertices = []
    
    # Iterate over axes (0, 1, 2 for X, Y, Z)
    for axis in range(3):
        # Iterate over each block type that needs meshing
        for block_type in meshable_block_types:
            # x_data maps p_index to [u32; 32] binary plane data
            x_data = {}
            
            # This loop structure implies iterating over existing p_indices
            # or generating them based on the voxel data. 
            # The source suggests iterating over p_index and data in x_data.
            # We assume x_data is populated elsewhere or via a nested loop over voxels.
            # For implementation completeness, we iterate over the keys of x_data.
            for p_index, data in x_data.items():
                # Extract AO from lower 9 bits
                ao = p_index & 0b111111111
                # Extract block_type from upper bits
                current_block_type = p_index >> 9
                
                # Generate quads for this binary plane
                quads = greedy_mesh_binary_plane(data, lod.size())
                
                # Append vertices for each quad
                for quad in quads:
                    append_vertices(vertices, face_dir, axis, lod, ao, current_block_type)
                    
    return vertices

def greedy_mesh_binary_plane(data: list, lod_size: int) -> list:
    """
    Generates quads from a binary plane represented by a list of u32.
    """
    greedy_quads = []
    for row in range(len(data)):
        y = 0
        while y < lod_size:
            # Find first solid voxel (skip zeros)
            # Note: trailing_zeros on a u32. If data[row] is 0, this might be undefined or large.
            # Assuming data[row] is treated as a bitfield.
            val = data[row]
            if val == 0:
                y += lod_size # Skip remaining
                break
                
            tz = val.trailing_zeros()
            y += tz
            
            if y >= lod_size:
                continue
                
            # Find height of solid run
            # Shift right by y to align the run to LSB
            shifted = val >> y
            # Count trailing ones
            h = shifted.trailing_ones()
            
            # Convert height to mask
            # 1 = 0b1, 2 = 0b11, etc.
            h_as_mask = (1 << h) - 1
            
            # Create quad from this run
            # Details of GreedyQuad creation depend on greedy algorithm specifics
            # Not fully detailed in source, but implied to be a standard greedy merge
            quad = create_quad_from_run(row, y, h, h_as_mask)
            greedy_quads.append(quad)
            
            y += h
            
    return greedy_quads

def append_vertices(vertices, face_dir, axis, lod, ao, block_type):
    """
    Appends vertices for a quad to the vertices list.
    Handles triangle winding and anisotropy flip.
    """
    # Placeholder for actual vertex generation logic based on quad and face_dir
    # The source mentions:
    # - Triangle vertex order depends on face_dir.reverse_order()
    # - Anisotropy flip if (v1ao > 0) ^ (v3ao > 0)
    # This function would generate the 4 vertices of the quad,
    # apply the flip, and add them to the vertices list.
    pass
```

**Parameters:**
- `lod.size()`: The dimension of the voxel chunk (e.g., 16, 32).
- `MESHABLE_BLOCK_TYPES`: List of block types to process.

**Edge Cases and Pitfalls:**
- **Precision:** AO values are packed into 9 bits. Ensure AO calculations fit within this range.
- **Boundaries:** The loop `y += (data[row] >> y).trailing_zeros()` must handle cases where `data[row]` is 0 to avoid infinite loops or undefined behavior.
- **Degenerate Input:** Empty chunks or chunks with no meshable blocks should return an empty vertex list.

## Subtopics

### Minecraft-style Voxel Meshing with AO and Flip
This variant generates mesh vertices for a specific face direction in a voxel chunk, calculating ambient occlusion (AO) values for the four corners of each face and determining a `flip_id` to optimize triangle winding. Unlike the standard multi-block approach, it explicitly handles face orientation and AO-based flipping to reduce overdraw and improve rendering performance. It is chosen when fine-grained control over vertex winding and AO-driven geometry optimization is required.

**Steps:**
1. For each voxel face (e.g., top face at y+1), calculate AO values for the four corners using `get_ao`.
2. Determine `flip_id`: `flip_id = ao[1] + ao[3] > ao[0] + ao[2]`.
3. Pack vertex data for the four corners (`v0, v1, v2, v3`) using `pack_data`, which includes world coordinates, `voxel_id`, face identifier, AO value, and `flip_id`.
4. If `flip_id` is true, add vertices to the index buffer in the order `v3, v1, v0, v3, v2, v1`. If false, add in the order `v0, v2, v1, v0, v3, v2`.

**Parameters:**
- Face identifier: Integer passed to `pack_data` to distinguish faces (e.g., 4 for top, 5 for front).

**Layout:**
- Vertex data is packed into a buffer via `add_data`. The specific layout of `packed_data` is defined by `pack_data` (not fully shown in this snippet, but implied to contain position, voxel_id, face_id, ao, flip_id).

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels and block types, as each block type is processed independently over the binary plane data.
- **Memory Complexity:** Proportional to the number of unique `p_index` values, as each requires a `[u32; 32]` array in `x_data`.
- **Performance:** Efficient for sparse voxel worlds due to binary plane representation and greedy meshing. AO calculation adds overhead but improves visual quality.
- **Strengths:** Supports multiple block types, preserves AO information, and reduces overdraw via flip optimization.
- **Limitations:** Complex packing of AO and block type into `p_index` may limit the range of AO values or block types. Requires careful handling of bitwise operations.

## Background
Multi-Block Binary Meshing with AO is an extension of binary greedy meshing techniques used in voxel-based games like Minecraft. It was developed to support multiple block types while maintaining the performance benefits of binary plane representation and AO. The method is commonly used in voxel engines that require detailed surface rendering with ambient occlusion.

## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
[S2] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
