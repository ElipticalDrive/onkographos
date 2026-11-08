# Multi-Block Binary Meshing with AO


## Overview
Multi-Block Binary Meshing with AO generates mesh vertices for a specific face direction by iterating over all defined block types and axes, handling ambient occlusion (AO) by packing it into the `p_index` [S1, unofficial]. It differs from its sibling [Greedy Quad Vertex Generation with AO and Anisotropy](4535-greedy-quad-vertex-generation-with-ao-and-anisotropy.md) by explicitly managing the iteration over block types and axes to support multiple material types within the same meshing pass [S1, unofficial].

## How it works
**Inputs:**
- `lod`: The voxel data structure containing the binary slices.
- `face_dir`: The direction of the face being generated (e.g., +X, -Y).
- `MESHABLE_BLOCK_TYPES`: A list of block types that require meshing.

**Outputs:**
- A vector of vertices, where each vertex contains position, AO value, and block type information.

**Data Structures:**
- `x_data`: A HashMap or array mapping `p_index` to `[u32; 32]`. The `p_index` encodes both the `ao` value and the `block_type`.
  - `ao`: Stored in the lower 9 bits (`p_index & 0b111111111`).
  - `block_type`: Stored in the upper bits (`p_index >> 9`).
- `GreedyQuad`: Represents a merged quad geometry.

**Steps:**
1. Initialize an empty `vertices` vector.
2. Iterate over each axis from 0 to `size` (the dimension of the voxel grid).
3. For each axis, iterate over each `block_type` in `MESHABLE_BLOCK_TYPES`.
4. Initialize `x_data` as a HashMap or array mapping `p_index` to `[u32; 32]`. If a `p_index` is not present, insert `[0u32; 32]`.
5. For each `p_index` and `data` in `x_data`:
   a. Extract `ao` from `p_index` using `p_index & 0b111111111`.
   b. Extract `block_type` from `p_index` using `p_index >> 9`.
   c. Call `greedy_mesh_binary_plane(data, lod.size())` to get a list of `GreedyQuad` objects.
6. For each `quad` returned:
   a. Call `append_vertices` with `vertices`, `face_dir`, `axis`, `lod`, `ao`, and `block_type`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_block_binary_meshing_with_ao(lod, face_dir, meshable_block_types):
    vertices = []
    
    # Iterate over each axis
    for axis in range(lod.size()):
        # Iterate over each block type
        for block_type in meshable_block_types:
            # Initialize x_data as a map from p_index to [u32; 32]
            x_data = {}
            
            # Populate x_data based on lod and block_type
            # This step involves extracting p_index (ao + block_type) and data for each slice
            # The exact extraction logic is implementation-specific but results in:
            # p_index = (block_type << 9) | ao
            # data = [u32; 32] representing the binary slice
            
            for p_index, data in x_data.items():
                # Extract ao and block_type from p_index
                ao = p_index & 0b111111111
                extracted_block_type = p_index >> 9
                
                # Generate quads for this binary plane
                quads = greedy_mesh_binary_plane(data, lod.size())
                
                # Append vertices for each quad
                for quad in quads:
                    append_vertices(vertices, face_dir, axis, lod, ao, extracted_block_type, quad)
                    
    return vertices

def append_vertices(vertices, face_dir, axis, lod, ao, block_type, quad):
    # Generate vertex positions based on quad geometry and lod
    # This step is implementation-specific but results in 4 vertices per quad
    v1, v2, v3, v4 = generate_quad_vertices(quad, face_dir, axis, lod)
    
    # Assign AO and block_type to vertices
    # The exact assignment depends on the vertex format
    # Typically, all vertices in a quad share the same ao and block_type
    # unless anisotropy or other factors dictate otherwise
    
    new_vertices = [v1, v2, v3, v4]
    
    # Adjust triangle vertex order based on face direction
    if face_dir.reverse_order():
        # Keep first index, reverse the rest
        o = new_vertices[1:]
        o.reverse()
        new_vertices = [new_vertices[0]] + o
        
    # Anisotropy flip
    # If v1ao > 0 XOR v3ao > 0, swap the first and last vertex
    # Note: In this method, ao is typically uniform per quad, so this might be a no-op
    # unless per-vertex ao is calculated later
    if (v1.ao > 0) ^ (v3.ao > 0):
        f = new_vertices.pop(0)
        new_vertices.append(f)
        
    vertices.extend(new_vertices)
```

**Edge Cases and Pitfalls:**
- Precision: Ensure that the `p_index` packing does not overflow the integer type used.
- Boundaries: Handle cases where `lod.size()` is 0 or negative.
- Degenerate Input: If `data` is all zeros, `greedy_mesh_binary_plane` should return an empty list of quads.

## Subtopics

### [Binary Greedy Voxel Meshing without Ambient Occlusion](4536-binary-greedy-voxel-meshing-without-ambient-occlusion.md)
This method generates mesh vertices for a face direction without ambient occlusion, iterating over block types and axes, using a fixed array for `x_data`. It outputs vertices with `ao=0` and `block_type=0`.

### [Minecraft-style Voxel Meshing with AO and Flip](4825-minecraft-style-voxel-meshing-with-ao-and-flip.md)
This method generates a mesh for a voxel segment by merging adjacent blocks of the same type into larger rectangular faces, reducing draw calls and vertex count. It extends binary greedy meshing by incorporating ambient occlusion (AO) values into the vertex data and using a "flip" bit to optimize triangle winding order based on AO distribution. It builds on [Multi-Block Binary Meshing with AO](4534-multi-block-binary-meshing-with-ao.md), which handles the iteration over block types and axes, by adding the specific logic for AO-based flip determination and vertex packing for the final geometry generation.

### [Greedy Lighting (AO Reconstruction)](10571-greedy-lighting-ao-reconstruction.md)
Greedy Lighting is a post-merge lighting computation method that reconstructs per-vertex ambient occlusion (AO) and light map values for large merged quads. It solves the problem of lighting artifacts that arise when multiple small blocks are merged into a single large face, ensuring that the lighting appearance remains correct by sampling neighboring block states. It differs from [Binary Greedy Voxel Meshing without Ambient Occlusion](4536-binary-greedy-voxel-meshing-without-ambient-occlusion.md) by explicitly calculating and blending AO values for each vertex of the merged quad based on the surrounding environment.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
