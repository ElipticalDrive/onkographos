# Binary Greedy Voxel Meshing without Ambient Occlusion
Also called: No-AO Binary Meshing
## Overview
This method generates mesh vertices for a face direction without ambient occlusion, iterating over block types and axes, using a fixed array for `x_data` [S1, unofficial]. It outputs vertices with `ao=0` and `block_type=0` [S1, unofficial].
## How it works
**Inputs:**
- `lod`: The voxel chunk data structure, providing access to block types and dimensions [S1, unofficial].
- `face_dir`: The normal direction of the face being generated [S1, unofficial].
- `axis`: The axis along which the iteration proceeds [S1, unofficial].

**Outputs:**
- A vector of mesh vertices [S1, unofficial].

**Data Structures:**
- `x_data`: A fixed array of 32 unsigned 32-bit integers (`[u32; 32]`) [S1, unofficial].
- `GreedyQuad`: A struct containing `x`, `y`, `w` (width), and `h` (height) of the generated quad [S1, unofficial].

**Algorithm Steps:**
1. Initialize an empty `vertices` vector [S1, unofficial].
2. Iterate over each `axis` from 0 to `size` [S1, unofficial].
3. Iterate over each `block_type` in `MESHABLE_BLOCK_TYPES` [S1, unofficial].
4. Initialize `x_data` as `[0u32; 32]` [S1, unofficial].
5. Call `greedy_mesh_binary_plane(x_data, lod.size())` to get quads [S1, unofficial].
6. For each quad, call `append_vertices` with `vertices`, `face_dir`, `axis`, `lod`, `ao=0`, `block_type=0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_greedy_voxel_meshing_no_ao(lod, face_dir):
    vertices = []
    for axis in range(3):
        for block_type in MESHABLE_BLOCK_TYPES:
            x_data = [0] * 32
            # The source notes do not specify the population logic for x_data.
            quads = greedy_mesh_binary_plane(x_data, lod.size())
            for quad in quads:
                append_vertices(vertices, face_dir, axis, lod, ao=0, block_type=0)
    return vertices

def greedy_mesh_binary_plane(data, lod_size):
    greedy_quads = []
    for row in range(len(data)):
        y = 0
        while y < lod_size:
            shifted = data[row] >> y
            tz = (shifted & -shifted).bit_length() - 1 if shifted else 32
            y += tz
            if y >= lod_size:
                continue
            shifted = data[row] >> y
            h = 0
            temp = shifted
            while temp & 1:
                h += 1
                temp >>= 1
            h_as_mask = (1 << h) - 1 if h < 32 else 0xFFFFFFFF
            mask = h_as_mask << y
            w = 1
            while row + w < lod_size:
                next_row_h = (data[row + w] >> y) & h_as_mask
                if next_row_h != h_as_mask:
                    break
                data[row + w] &= ~mask
                w += 1
            greedy_quads.append({'x': row, 'y': y, 'w': w, 'h': h})
    return greedy_quads
```

**Edge Cases and Pitfalls:**
- **Precision:** The `x_data` array is fixed at 32 bits. If `lod.size()` exceeds 32, the algorithm as described may fail or require modification [S1, unofficial].
## Subtopics

### [Binary Greedy Meshing with Transparency Support](4839-binary-greedy-meshing-with-transparency-support.md)
Extends this method to handle transparent blocks by generating face masks that include transparent voxels adjacent to solid ones, preserving geometry for depth-tested rendering pipelines.

### [Binary Greedy Voxel Meshing with Ambient Occlusion](4536-binary-greedy-voxel-meshing-with-ambient-occlusion.md)
Adds AO computation by packing AO values into the vertex index, providing shading information.

### [Streaming Binary Greedy Meshing](4535-streaming-binary-greedy-meshing.md)
Optimizes memory usage by processing chunks in a streaming fashion.

### [Multi-Block Binary Meshing with AO](4534-multi-block-binary-meshing-with-ao.md)
The parent method that handles multiple block types and axes with AO support.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.
## Background
- **Origin:** Not given in the sources.
- **History and development:** Not given in the sources.
- **Usage:** Not given in the sources.
## Sources
[S1] Binary Greedy Voxel Meshing Algorithm, https://github.com/TanTanDev/binary_greedy_mesher_demo, fetched 2026-10-26 (unofficial)
