# Voxel Face Mesh Generation

Also called: Block Face Culling, Visible Block Faces Meshing, Voxel Quad Generation

## Overview
Voxel Face Mesh Generation is a surface extraction method that constructs a single unit quad mesh for every exposed face of a voxel block, populating a vertex buffer with positions, indices, normals, UVs, tangents, and colors based on face direction and block type [S1, unofficial]. It solves the problem of converting discrete voxel data into opaque geometry suitable for standard GPU pipelines by emitting one quad per visible block face in a padded chunk [S2, unofficial]. It differs from [Greedy face merging](8935-greedy-face-merging.md) by not merging adjacent coplanar faces, resulting in higher vertex counts but simpler logic and easier UV mapping per block type [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel chunk data structure containing block types and visibility states, padded by 1 voxel on all sides to allow neighbor lookups without bounds checks [S1, S2, unofficial].
*   **Output:** A `MeshData` structure containing dynamic arrays for vertices, triangle indices, normals, UVs, tangents, and vertex colors [S1, unofficial]; or an intermediate array of 6 vectors of `Quad` structs indexed by face direction [S2, unofficial].

**Data Structures:**
*   `MeshData` (from [S1, unofficial]):
    *   `vertices`: Dynamic array of `FVector` (position).
    *   `Triangles`: Dynamic array of `int` (indices).
    *   `normals`: Dynamic array of `FVector`.
    *   `UV0`: Dynamic array of `FVector2D`.
    *   `tangents`: Dynamic array of `FProcMeshTangent`.
    *   `vertexColors`: Dynamic array of `FLinearColor`.
    *   `VERTICES[6][4]`: Static lookup table of base quad vertices for each of the 6 face directions.
    *   `NORMALS[6]`: Static lookup table of normals.
    *   `UVS_Inverted[6][4]`: Static lookup table of inverted base UVs.
    *   `TANGENTS[6]`: Static lookup table of tangents.
    *   `BlockTypeTextureIndex[MAX_TYPES][6]`: Lookup table mapping block type and face direction to texture atlas index.
    *   `ATLAS_SIZE`: Float constant defining texture atlas resolution.
*   `Quad` (from [S2, unofficial]):
    *   `voxel`: Array `[x, y, z]` of voxel coordinates.
    *   `width`: Integer, typically 1.
    *   `height`: Integer, typically 1.
    *   Output is an array `[Vec<Quad>; 6]` indexed by `FaceDir` discriminant.

**Parameters:**
*   `BlockSize`: Voxel world unit size, used to scale vertex positions [S1, unofficial].
*   `ATLAS_SIZE`: Texture atlas resolution, used to compute `UVSize = 1.0f / ATLAS_SIZE` [S1, unofficial].
*   `flip_v`: Boolean flag; `true` for side faces (XNeg, XPos, ZNeg, ZPos), `false` for top/bottom faces [S2, unofficial].

**Steps:**
1.  **Iterate Voxels:** Loop through interior voxels only, using indices from 1 to `PADDED_CHUNK_MAX_INDEX` exclusive, to leverage the 1-voxel padding for neighbor lookups [S1, S2, unofficial].
2.  **Check Visibility:** For each non-empty voxel, iterate all 6 `FaceDir` directions. Calculate the neighbor's linear index using the face normal: `neighbor_idx = padded_linearize([x + nx, y + ny, z + nz])` [S2, unofficial]. Determine if the face needs meshing using the visibility rule [S2, unofficial]:
    *   `face_needs_mesh(voxel, neighbor)` returns `true` if: `(voxel is Opaque AND neighbor is not Empty) OR (voxel is Translucent AND neighbor is Empty)`.
    *   It returns `false` if: `(voxel is Empty) OR (neighbor is Opaque) OR (both are Translucent)` [S2, unofficial].
3.  **Generate Geometry:** For each face that needs meshing:
    *   **Position:** Calculate vertex position offset: `position = FVector(k * BlockSize, j * BlockSize, i * BlockSize)` [S1, unofficial]. Add 4 vertices to the buffer by scaling base quad vertices from `VERTICES[direction]` by 100 and adding the position [S1, unofficial].
    *   **Indices:** Add 6 triangle indices forming two triangles for the quad: `(0,1,2)` and `(3,2,1)` relative to the last vertex count. The winding order `(3,2,1)` for the second triangle maintains consistent face culling orientation [S1, unofficial].
    *   **Normals:** Add 4 normals by repeating `NORMALS[direction]` four times [S1, unofficial].
    *   **Tangents:** Add 4 tangents by repeating `TANGENTS[direction]` four times [S1, unofficial].
    *   **Colors:** Add 4 vertex colors hardcoded to `FLinearColor(0.75, 0.75, 0.75, 1.0)` [S1, unofficial].
    *   **UVs:** Compute UV atlas coordinates: `UVI = index / ATLAS_SIZE`, `UVJ = index % ATLAS_SIZE`, `UVSize = 1.0f / ATLAS_SIZE` [S1, unofficial]. The texture index is looked up from `BlockTypeTextureIndex[blockType][direction]`, where `UP->0`, `DOWN->2`, and `RIGHT/LEFT/FORWARD/BACK->1` [S1, unofficial]. Add 4 UVs using inverted base UVs scaled by `UVSize` and offset by `UVJ/UVI` atlas coordinates. The UV mapping uses inverted base UVs and swaps U/V components via `FVector2D(UVJ * UVSize, UVI * UVSize)` to align with the atlas layout [S1, unofficial]. For side faces, `flip_v` is applied [S2, unofficial].
    *   **Texture Fallback:** For opaque style, look up texture in `block_textures` using `chunk_block_id`, `block_face`, and `face_uvs`. If texture exists, use its coords; if color, use raw RGBA; if missing, use black UVs/colors [S2, unofficial]. For grass/snow-style blocks, if the voxel directly above is opaque, the block face is set to `Face::Bottom` to use the bottom texture instead of the side texture [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_face_mesh(chunk_data, block_size, atlas_size):
    """
    Generates a mesh from voxel chunk data using face culling.
    chunk_data: 3D array of block types, padded by 1 on all sides.
    block_size: float, size of one voxel unit.
    atlas_size: float, resolution of the texture atlas.
    """
    vertices = []
    indices = []
    normals = []
    uvs = []
    tangents = []
    colors = []
    
    # Base quad vertices for 6 directions (pre-defined in source [S1, unofficial])
    # VERTICES[direction][4]
    # NORMALS[direction]
    # TANGENTS[direction]
    # BlockTypeTextureIndex[block_type][direction]
    
    last_vertex_count = 0
    
    # Iterate interior voxels only (indices 1 to MAX-1) to use padding
    for i in range(1, chunk_data.height - 1):
        for j in range(1, chunk_data.width - 1):
            for k in range(1, chunk_data.depth - 1):
                block_type = chunk_data[i, j, k]
                
                # Skip empty voxels
                if block_type == BlockType.AIR:
                    continue
                
                # Check all 6 faces
                for direction in FaceDir.ALL:
                    # Calculate neighbor position
                    nx, ny, nz = direction.normal
                    neighbor_pos = [i + nx, j + ny, k + nz]
                    neighbor_type = chunk_data[neighbor_pos[0], neighbor_pos[1], neighbor_pos[2]]
                    
                    # Visibility rule [S2, unofficial]
                    if not face_needs_mesh(block_type, neighbor_type):
                        continue
                    
                    # Position [S1, unofficial]
                    position = [k * block_size, j * block_size, i * block_size]
                    base_vertices = VERTICES[direction] # 4 vertices
                    
                    for v in range(4):
                        # Scale by 100 and add position [S1, unofficial]
                        verts = [base_vertices[v][0] * 100 + position[0],
                                 base_vertices[v][1] * 100 + position[1],
                                 base_vertices[v][2] * 100 + position[2]]
                        vertices.append(verts)
                    
                    # Indices [S1, unofficial]
                    indices.append(last_vertex_count + 0)
                    indices.append(last_vertex_count + 1)
                    indices.append(last_vertex_count + 2)
                    indices.append(last_vertex_count + 3)
                    indices.append(last_vertex_count + 2)
                    indices.append(last_vertex_count + 1)
                    
                    # Normals [S1, unofficial]
                    normal = NORMALS[direction]
                    for _ in range(4):
                        normals.append(normal)
                    
                    # Tangents [S1, unofficial]
                    tangent = TANGENTS[direction]
                    for _ in range(4):
                        tangents.append(tangent)
                    
                    # Colors [S1, unofficial]
                    color = [0.75, 0.75, 0.75, 1.0]
                    for _ in range(4):
                        colors.append(color)
                    
                    # UVs [S1, unofficial]
                    # Get texture index for block type and direction
                    tex_index = BlockTypeTextureIndex[block_type][direction]
                    uvi = tex_index // atlas_size
                    uvj = tex_index % atlas_size
                    uv_size = 1.0 / atlas_size
                    
                    # Inverted base UVs with swap [S1, unofficial]
                    base_uvs = UVS_Inverted[direction] # 4 UVs
                    for v in range(4):
                        # Swap U/V and scale/offset
                        u = base_uvs[v][1] * uv_size + uvj * uv_size
                        v_coord = base_uvs[v][0] * uv_size + uvi * uv_size
                        uvs.append([u, v_coord])
                    
                    last_vertex_count += 4
    
    return {
        'vertices': vertices,
        'indices': indices,
        'normals': normals,
        'uvs': uvs,
        'tangents': tangents,
        'colors': colors
    }

def face_needs_mesh(voxel_type, neighbor_type):
    """Visibility rule [S2, unofficial]"""
    if voxel_type == BlockType.AIR:
        return False
    if neighbor_type == BlockType.OPAQUE:
        return False
    if voxel_type == BlockType.TRANSLUCENT and neighbor_type == BlockType.AIR:
        return True
    if voxel_type == BlockType.OPAQUE and neighbor_type != BlockType.AIR:
        return True
    if voxel_type == BlockType.TRANSLUCENT and neighbor_type == BlockType.TRANSLUCENT:
        return False
    # Default fallback for other combinations not explicitly covered
    return False
```

## Subtopics

### [Texture Atlas UV Mapping](4238-texture-atlas-uv-mapping.md)
Texture Atlas UV Mapping assigns UV coordinates to voxel face meshes by mapping block types to specific sub-regions of a shared texture atlas, enabling efficient single-material rendering of complex voxel worlds. It differs from per-vertex UV generation by using a per-face lookup strategy that accounts for block type and face orientation, ensuring correct texture alignment and winding order across the mesh.

### [Face Direction and Quad Geometry](4881-face-direction-and-quad-geometry.md)
This method defines the geometric primitives for voxel face meshing by mapping six axis-aligned face directions to explicit vertex positions, indices, and UV coordinates. It solves the problem of converting abstract voxel face data into a renderable triangle strip or indexed triangle list suitable for standard GPU pipelines. It differs from [Texture Atlas UV Mapping](4238-texture-atlas-uv-mapping.md) by handling the raw spatial positioning and winding order logic, whereas the sibling method handles texture coordinate lookup and packing.

## Operating characteristics
**Time and Memory Complexity:**
*   Time complexity is proportional to the number of exposed faces, which in the worst case (checkerboard pattern) is $O(N)$ where $N$ is the number of voxels, but typically much lower for solid structures.
*   Memory complexity is proportional to the number of generated quads, storing 4 vertices and 6 indices per quad, plus normals, tangents, UVs, and colors [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Scales linearly with the number of exposed faces. Chunk-based processing allows for parallel generation [S1, S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Simple implementation, easy to map textures per block type, handles transparency logic explicitly [S1, S2, unofficial].
*   **Limitations:** High vertex count compared to greedy meshing, leading to larger buffers and potentially lower draw call efficiency if not batched [S1, S2, unofficial].

**When to use it and when not:**
*   Use when simplicity of implementation and per-block-type texture mapping are prioritized over mesh density [S1, S2, unofficial].
*   Do not use when maximum mesh compression and draw call reduction are critical, in which case [Greedy face merging](8935-greedy-face-merging.md) is preferred.

## Background
Voxel Face Mesh Generation is a common technique in voxel game development, particularly in engines like Unreal Engine for Minecraft-like clones [S1, unofficial]. It is also used in Rust-based voxel engines like Infinigen [S2, unofficial]. The method relies on face culling to determine which faces of a voxel block are visible and should be included in the mesh [S1, S2, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S2] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
