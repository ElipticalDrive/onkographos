# Voxel Face Mesh Generation

Also called: Exposed Face Mesh Generation, Face-to-Vertex Generation with UV and Lighting, Visible Block Faces Meshing, Voxel Quad Generation

## Overview
Voxel Face Mesh Generation is a surface extraction algorithm that constructs a triangle mesh from a 3D voxel grid by emitting a single unit quad for every face of a solid voxel that is exposed to empty space or a chunk boundary [S1, S3, S5, unofficial]. It solves the problem of rendering large voxel worlds efficiently by culling internal faces between adjacent solid voxels, thereby reducing vertex count and draw calls compared to naive voxel rendering [S1, S9, unofficial]. Unlike smooth surface extraction methods like Marching Cubes, this method produces a blocky, axis-aligned aesthetic suitable for voxel-art styles [S1, unofficial][S8]. It differs from [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by focusing on simple neighbor-based face culling rather than combining multiple voxel sets via boolean operations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid (occupancy or density field), typically stored as a padded chunk to simplify boundary checks [S3, S5, S9, unofficial]. The grid contains block types or density values.
*   **Output:** A mesh data structure containing arrays for vertices (positions, normals, UVs, tangents, colors), indices (triangle connectivity), and UV coordinates [S2, S7, unofficial].

**Data Structures:**
*   **Voxel Grid:** Stored as a 1D array or 3D array. For example, a 1D byte array `voxelMap` of size `ChunkWidth * ChunkHeight * ChunkSize`, indexed by `x + Width * (y + Height * z)` [S7, unofficial]. Alternatively, a density field with an `iso_value` threshold [S5, unofficial].
*   **MeshData:** Holds dynamic arrays for `vertices` (position, normal, UV, tangent, color), `triangles` (indices), and `uvs` [S2, S7, unofficial].
*   **Lookup Tables:** Static arrays defining base geometry for the 6 face directions: `VERTICES[6][4]` (base quad corners), `NORMALS[6]` (face normals), `TANGENTS[6]`, and `UVS_Inverted[6][4]` [S2, unofficial]. Texture atlas indices are mapped via `BlockTypeTextureIndex[MAX_TYPES][6]` [S2, unofficial].

**Steps:**
1.  **Initialization:** Clear mesh buffers (`vertices`, `triangles`, `uvs`, `normals`) and pre-allocate capacity based on estimated visible face count (e.g., `ChunkWidth^2 * 6 * 4` vertices) [S7, S9, unofficial].
2.  **Iteration:** Loop through all voxels in the chunk. If using padding, iterate only interior voxels (indices 1 to `PADDED_CHUNK_MAX_INDEX`) to avoid boundary checks [S3, unofficial]. Skip empty voxels (e.g., `BlockType.Air` or density < `iso_value`) [S5, S7, unofficial].
3.  **Neighbor Check:** For each solid voxel, check its 6 immediate neighbors (±X, ±Y, ±Z) [S1, S9, unofficial].
    *   If a neighbor is out of chunk bounds, treat it as empty/visible [S9, unofficial].
    *   If a neighbor is empty (Air, density < `iso_value`, or `visible=false`), the face is exposed [S1, S5, unofficial].
    *   *Translucency Rule:* For translucent voxels, a face is generated if the neighbor is empty; if the neighbor is opaque, the face is hidden [S3, unofficial].
4.  **Face Generation:** For each exposed face:
    *   **Vertices:** Calculate 4 vertex positions by adding the voxel's world position (scaled by `BlockSize`) to the base quad vertices defined for the face direction [S2, S7, unofficial].
    *   **Indices:** Add 6 indices to form two triangles. The winding order must be consistent (e.g., counter-clockwise). A common pattern is `(0,1,2)` and `(3,2,1)` relative to the current vertex count to ensure correct culling [S2, S7, unofficial].
    *   **Normals:** Assign the face normal (from `NORMALS[direction]`) to all 4 vertices [S2, unofficial].
    *   **UVs:** Compute UV coordinates based on the block type and face direction using a texture atlas. Calculate atlas tile indices (`UVI`, `UVJ`) and scale by `UVSize = 1.0 / ATLAS_SIZE`. Apply UV inversion or swapping if required by the atlas layout [S2, unofficial].
    *   **Colors/Lighting:** Assign vertex colors. This can be a flat color based on block type [S6, unofficial] or per-vertex lighting colors (sun/block light) sampled from neighbors for smooth lighting [S1, S2, unofficial].
5.  **Finalization:** Upload the accumulated vertex and index data to the GPU mesh object. Recalculate normals, bounds, and tangents if not computed during generation [S1, S7, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_mesh(chunk_data, chunk_width, chunk_height, block_size, atlas_size):
    """
    Generates a mesh from a voxel chunk by emitting quads for exposed faces.
    chunk_data: 1D array of block IDs or densities.
    Returns: vertices, indices, uvs, normals, colors.
    """
    vertices = []
    indices = []
    uvs = []
    normals = []
    colors = []
    
    # Base quad vertices for a unit cube at origin, indexed by face direction (0-5)
    # Example: VERTICES[direction][vertex_index] -> (x, y, z)
    # Directions: UP, DOWN, NORTH, SOUTH, EAST, WEST
    base_vertices = get_base_quad_vertices() 
    base_normals = get_base_normals()
    
    vertex_count = 0
    
    # Iterate through all voxels
    for z in range(chunk_height):
        for y in range(chunk_width):
            for x in range(chunk_width):
                block_id = chunk_data[x + y * chunk_width + z * chunk_width * chunk_height]
                
                # Skip empty/air blocks
                if is_empty(block_id):
                    continue
                
                # Check 6 neighbors
                for face_dir in range(6):
                    neighbor_pos = get_neighbor_pos(x, y, z, face_dir)
                    
                    # Check if neighbor is out of bounds or empty
                    is_exposed = False
                    if not is_in_bounds(neighbor_pos, chunk_width, chunk_height):
                        is_exposed = True
                    else:
                        neighbor_id = chunk_data[
                            neighbor_pos.x + neighbor_pos.y * chunk_width + neighbor_pos.z * chunk_width * chunk_height
                        ]
                        if is_empty(neighbor_id):
                            is_exposed = True
                        # Optional: Handle translucency rules here
                        # if is_translucent(block_id) and not is_empty(neighbor_id):
                        #     is_exposed = False
                    
                    if is_exposed:
                        # Generate Quad
                        # 1. Vertices
                        for v_idx in range(4):
                            base_v = base_vertices[face_dir][v_idx]
                            world_pos = (base_v * block_size) + (Vector3(x, y, z) * block_size)
                            vertices.append(world_pos)
                            
                            # 2. Normal
                            normals.append(base_normals[face_dir])
                            
                            # 3. UVs
                            uv = compute_uv(block_id, face_dir, atlas_size)
                            uvs.append(uv)
                            
                            # 4. Color (Flat or Smooth)
                            color = compute_color(block_id, face_dir, chunk_data, x, y, z)
                            colors.append(color)
                        
                        # 5. Indices (Two triangles: 0-1-2 and 3-2-1 for consistent winding)
                        indices.append(vertex_count + 0)
                        indices.append(vertex_count + 1)
                        indices.append(vertex_count + 2)
                        indices.append(vertex_count + 3)
                        indices.append(vertex_count + 2)
                        indices.append(vertex_count + 1)
                        
                        vertex_count += 4
                        
    return vertices, indices, uvs, normals, colors
```

**Parameters:**
*   `BlockSize`: Scale factor for vertex positions [S2, unofficial].
*   `AtlasSize`: Resolution of the texture atlas, used to compute `UVSize = 1.0 / AtlasSize` [S2, unofficial].
*   `IsoValue`: Threshold for density fields; voxels with density >= `IsoValue` are solid [S5, unofficial].
*   `ChunkSize`: Dimensions of the voxel chunk (e.g., 16x64x16) [S9, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Checks:** Chunk boundaries must be explicitly handled to generate faces at the edge of the world or chunk [S9, unofficial]. If not padded, neighbor lookups require bounds checking [S3, unofficial].
*   **Winding Order:** Incorrect triangle winding (e.g., both triangles clockwise) causes back-face culling to hide geometry [S2, unofficial].
*   **UV Inversion:** UVs may need to be flipped or swapped (U/V) to align with the texture atlas layout for specific face directions [S2, unofficial].
*   **Memory Leaks:** In editor environments (e.g., Unity), modifying `SharedMesh` is required to avoid asset duplication; runtime uses `Mesh` directly [S1, unofficial].
*   **Cross-Chunk Visibility:** If chunks are not processed together, internal faces at chunk boundaries may be incorrectly generated or hidden. Neighbor queries across chunks must query adjacent chunk data [S7, S9, unofficial].

## Subtopics

### [Texture Atlas UV Mapping](4238-texture-atlas-uv-mapping.md)
Maps block types to sub-regions of a shared texture atlas using a per-face lookup table, with UV inversion for correct winding order. It assigns UV coordinates to voxel face meshes based on block type and face orientation, enabling efficient single-material rendering of complex voxel worlds.

### [Face Direction and Quad Geometry](4881-face-direction-and-quad-geometry.md)
Defines the six axis-aligned face orientations on a voxel cube and provides the deterministic logic for generating quad geometry (positions, indices, UVs, and normals) for meshing. It solves the problem of converting abstract voxel face exposure data into concrete vertex buffer layouts suitable for standard rasterization pipelines.

### [Block Face Mesh Generation](10913-block-face-mesh-generation.md)
Generates a 6-vertex mesh data structure for a single exposed face of a voxel block, applying special geometry for cactus blocks and packing vertex attributes for GPU consumption. It differs from its sibling [Face Direction and Quad Geometry](4881-face-direction-and-quad-geometry.md) by handling block-specific geometric modifications (like cactus insets) and producing a packed vertex format rather than just raw geometric primitives. It serves as the core geometry generation step within [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md).

### [Voxel Meshing with Smooth Lighting](10951-voxel-meshing-with-smooth-lighting.md)
Voxel Meshing with Smooth Lighting is a surface extraction algorithm for voxel chunks that generates a single quad mesh for exposed voxel faces, populating a `MeshData` buffer with vertices, indices, normals, UVs, and vertex colors based on face direction and block type. It builds on [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) by extending the vertex data to include per-vertex lighting colors (sun and block light) and supports smooth lighting by sampling 8 neighboring voxels per face to create smoother lighting transitions. Unlike flat lighting which uses a single neighbor value, this method averages light from surrounding voxels to reduce the "blocky" appearance of shadows.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the chunk. Each voxel is processed once, and up to 6 faces are checked.
*   **Memory Complexity:** $O(F)$, where $F$ is the number of exposed faces. The output mesh size is proportional to the surface area of the solid voxels.
*   **Performance:** Significantly faster than greedy meshing or Marching Cubes for simple voxel worlds [S8]. It produces many more quads than greedy meshing (which merges adjacent coplanar faces) but is simpler to implement and update incrementally [S8].
*   **Strengths:** Simple implementation, easy to update incrementally (only regenerate affected chunks/voxels), low memory overhead for dynamic worlds.
*   **Limitations:** High vertex count compared to optimized meshing techniques (greedy, quadric), blocky aesthetic not suitable for smooth terrain, requires careful handling of chunk boundaries for seamless worlds.
*   **When to Use:** Voxel-based games (e.g., Minecraft clones), real-time terrain editing, simple procedural generation.
*   **When Not to Use:** High-fidelity smooth terrain, scenes requiring minimal draw calls where greedy meshing is preferred.

## Background
Voxel Face Mesh Generation is a foundational technique in voxel rendering, originating from early voxel engines and popularized by games like Minecraft. It relies on the principle of face culling, where internal faces between adjacent solid voxels are omitted to save rendering time [S1, unofficial]. The method is widely used in Unity and Unreal Engine voxel tutorials and implementations [S1, S2, unofficial]. It serves as the basis for more advanced techniques like smooth lighting and texture atlas mapping [S1, S2, unofficial].

## Sources
[S1] Voxels #5 - A Procedural Mesh of Voxels, https://www.youtube.com/watch?v=014EbVkuHSM, fetched 2026-10-27 (unofficial)
[S2] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S3] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S4] afonsolage/projekto, https://github.com/afonsolage/projekto, fetched 2026-10-27 (unofficial)
[S5] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
[S6] Shilghter/VoxelTerrain, https://github.com/Shilghter/VoxelTerrain, fetched 2026-10-27 (unofficial)
[S7] philip2012/voxel-forge, https://github.com/philip2012/voxel-forge, fetched 2026-10-27 (unofficial)
[S8] visible_block_faces fn block_mesh::visible_block_faces, https://docs.rs/block-mesh/latest/block_mesh/fn.visible_block_faces.html, fetched 2026-10-27
[S9] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
