# Voxel Mesh Generation with Face Culling

Also called: Chunk-based Mesh Generation, Face Culling & Chunk Mesh Generation, Face Culling Voxel Meshing, Greedy Meshing (Basic), Voxel Chunk Mesh Generation with Adjacency Culling, Voxel Chunk Mesh Generation with Face Culling, Voxel Mesh Generation (Face Culling), Voxel Surface Extraction

## Overview

Voxel Mesh Generation with Face Culling is a CPU-based surface extraction method that converts a 3D voxel grid into a static triangle mesh by iterating through voxels and emitting geometry only for faces adjacent to empty space (air) or transparent blocks [S2, S3, S4]. It solves the problem of excessive vertex and index counts in naive voxel rendering by hiding internal faces shared between two solid blocks [S4]. Unlike [Naive Voxel Meshing](2376-naive-voxel-meshing.md), which renders every voxel as a full cube, this method reduces draw calls and vertex count by culling occluded geometry [S4]. It differs from [Visible Block Faces](2268-visible-block-faces.md) by typically operating on fixed-size chunks with explicit adjacency bitmasks rather than purely visibility-driven culling per frame, and from [GPU Marching Cubes (Face-based)](3674-gpu-marching-cubes-face-based.md) by being a deterministic, static mesh generation approach suitable for CPU-side updates [S1, S3, S4].

## How it works

### Inputs and Outputs
- **Input**: A 3D voxel grid representing a chunk of world data. The grid is typically organized as a 1D array or 3D array of block IDs (e.g., `u8`), where `0` represents air and values $>0$ represent solid or transparent blocks [S3, S4].
- **Output**: A vertex buffer and an index buffer containing a triangle mesh representing the visible surface of the chunk [S3, S4].

### Data Structures

**Voxel Grid Layout**:
The voxel data is stored in a 1D array `voxels` of size $W \times H \times D$, where $W$ is chunk width, $H$ is chunk height, and $D$ is chunk depth [S3]. The 3D coordinate $(x, y, z)$ maps to the 1D index using the formula:
$$ \text{index} = x \cdot H \cdot D + y \cdot D + z $$
where $0 \le x < W$, $0 \le y < H$, and $0 \le z < D$ [S3].

**Adjacency Flags (Optional Optimization)**:
Some implementations use a `ChunkSidesArray` of type `[[[u8; CHUNK_SIZE]; CHUNK_SIZE]; CHUNK_SIZE]` [S1]. Each `u8` at index `[i][j][k]` stores adjacency flags for the block at that position, where a bit is set if the adjacent block in that direction is non-opaque (visible) and cleared if opaque (occluded) [S1].

**Face Geometry Constants**:
- **VERTICES**: An array `[[f32; 3]; 8]` defining the 8 corners of a unit cube at $(0,0,0)$ to $(1,1,1)$ [S1].
- **FACES**: An array `[[usize; 4]; 6]` mapping each of the 6 face indices to the 4 vertex indices forming the quad [S1].
- **NORMALS**: An array `[[f32; 3]; 6]` defining the outward normal for each face [S1].
- **FACE_ORDER**: An array `[usize; 6]` defining the triangle fan or index order for rendering [S1].
- **UVS**: An array `[[f32; 2]; 4]` defining the unit square UV coordinates for a face [S1].

**Texture Atlas Mapping**:
A texture atlas is used to map block types to UV coordinates. For example, with `ATLAS_WIDTH_PX = 256`, `ATLAS_HEIGHT_PX = 256`, and `TILE_SIZE_PX = 16`:
- Block 1 (Grass): Top face (4) -> row 12, col 12; Bottom face (5) -> row 0, col 2; Sides (0-3) -> row 0, col 3 [S4].
- Block 2 (Dirt): All faces -> row 0, col 2 [S4].
- Block 3 (Stone): All faces -> row 0, col 1 [S4].
- Default/Error: row 9, col 9 [S4].

### Parameters
- **CHUNK_SIZE**: The spatial dimension of a chunk in blocks. Common values are 16 [S3, S4] or 32 [S2]. Must be a multiple of 8 in some implementations [S1].
- **CHUNK_HEIGHT**: The height of the chunk in voxels. Common values are 64 [S3, S4].
- **ATLAS_WIDTH_PX**: Texture atlas width in pixels (e.g., 256.0) [S4].
- **ATLAS_HEIGHT_PX**: Texture atlas height in pixels (e.g., 256.0) [S4].
- **TILE_SIZE_PX**: Size of a single texture tile in pixels (e.g., 16.0) [S4].
- **ATLAS_ROWS**: Number of rows in the texture atlas (e.g., 16) [S4].
- **ATLAS_COLS**: Number of columns in the texture atlas (e.g., 16) [S4].

### Algorithm Steps

1. **Initialize Buffers**: Create empty vertex and index buffers.
2. **Iterate Voxels**: Loop through each voxel $(x, y, z)$ in the chunk, where $0 \le x < W$, $0 \le y < H$, $0 \le z < D$ [S2, S3, S4].
3. **Check Solidity**: Retrieve the block ID at $(x, y, z)$. If the block is air (ID == 0) or invisible (if `isInvisible` flag is true), skip to the next voxel [S2, S3, S4].
4. **Check Neighbors**: For each of the 6 faces (front, back, left, right, top, bottom), determine the neighbor's coordinates $(nx, ny, nz)$ [S4].
   - Neighbor offsets: Front $(0,0,1)$, Back $(0,0,-1)$, Left $(-1,0,0)$, Right $(1,0,0)$, Top $(0,1,0)$, Bottom $(0,-1,0)$ [S4].
   - **Boundary Check**: If the neighbor coordinates are out of bounds (e.g., $nx < 0$ or $nx \ge W$), treat the neighbor as air (empty space) [S3, S4].
   - **Visibility Check**: If the neighbor is not solid (isSolid returns false), the face is visible [S4].
5. **Generate Face Geometry**: If the face is visible:
   - **Calculate Vertex Positions**: For each of the 4 vertices of the face quad, calculate the final position by adding the chunk offset $(x, y, z)$ to the unit cube vertex coordinates [S1, S4].
     $$ \text{pos}_i = \text{VERTICES}[\text{FACES}[\text{face}][i]] + [x, y, z] $$
   - **Calculate UVs**: Determine the UV coordinates based on the block type and face orientation using the texture atlas mapping [S4].
     $$ \text{finalU} = (\text{pngCol} \cdot \text{texStepX}) + (\text{localU} \cdot \text{texStepX}) $$
     $$ \text{finalV} = (\text{glRow} \cdot \text{texStepY}) + (\text{localV} \cdot \text{texStepY}) $$
     where $\text{texStepX} = \text{TILE\_SIZE\_PX} / \text{ATLAS\_WIDTH\_PX}$ and $\text{texStepY} = \text{TILE\_SIZE\_PX} / \text{ATLAS\_HEIGHT\_PX}$ [S4].
   - **Calculate Normals**: Retrieve the normal for the face from the `NORMALS` array [S1].
   - **Add Vertices**: Push a vertex struct containing position, UV, and normal to the vertex buffer [S1, S4].
   - **Add Indices**: Add 6 indices (2 triangles) to the index buffer, relative to the current end of the vertex list [S3].
6. **Bake Lighting (Optional)**: If baked lighting is used, modify vertex colors based on face orientation: top faces full brightness, side faces darkened by a fixed multiplier, bottom faces darker still [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(chunk_voxels, chunk_width, chunk_height, chunk_depth, texture_atlas):
    vertices = []
    indices = []
    
    # Define face normals and vertex offsets for a unit cube
    # FACES: list of 6 faces, each face has 4 vertex indices
    # VERTICES: list of 8 vertex positions for a unit cube
    # NORMALS: list of 6 normals for each face
    # UVS: list of 4 UVs for each face
    
    for x in range(chunk_width):
        for y in range(chunk_height):
            for z in range(chunk_depth):
                block_id = chunk_voxels[x * chunk_height * chunk_depth + y * chunk_depth + z]
                
                # Skip air or invisible blocks
                if block_id == 0 or is_invisible(block_id):
                    continue
                
                # Check all 6 faces
                for face_idx in range(6):
                    # Determine neighbor coordinates
                    nx, ny, nz = get_neighbor_coords(x, y, z, face_idx)
                    
                    # Check if neighbor is out of bounds or not solid
                    if is_out_of_bounds(nx, ny, nz, chunk_width, chunk_height, chunk_depth):
                        is_visible = True
                    else:
                        neighbor_id = chunk_voxels[nx * chunk_height * chunk_depth + ny * chunk_depth + nz]
                        is_visible = not is_solid(neighbor_id)
                    
                    if is_visible:
                        # Generate 4 vertices for the face
                        for v_idx in range(4):
                            # Get unit vertex position
                            unit_pos = VERTICES[FACES[face_idx][v_idx]]
                            
                            # Add chunk offset
                            final_pos = [unit_pos[0] + x, unit_pos[1] + y, unit_pos[2] + z]
                            
                            # Get UV coordinates
                            uv = UVS[face_idx][v_idx]
                            # Apply texture atlas mapping
                            atlas_u, atlas_v = get_atlas_uv(block_id, face_idx, uv)
                            
                            # Get normal
                            normal = NORMALS[face_idx]
                            
                            # Add vertex
                            vertices.append(Vertex(final_pos, atlas_u, atlas_v, normal))
                        
                        # Add 6 indices for 2 triangles
                        base_idx = len(vertices) - 4
                        indices.extend([base_idx, base_idx + 1, base_idx + 2])
                        indices.extend([base_idx, base_idx + 2, base_idx + 3])
    
    return vertices, indices

def is_out_of_bounds(x, y, z, w, h, d):
    return x < 0 or x >= w or y < 0 or y >= h or z < 0 or z >= d

def is_solid(block_id):
    return block_id != 0 and not is_invisible(block_id)

def is_invisible(block_id):
    # Implementation depends on block type
    return False # Placeholder
```

### Edge Cases and Pitfalls
- **Boundary Checks**: The `is_solid` function must explicitly handle chunk boundaries by returning false for out-of-bounds coordinates, treating out-of-bounds space as air [S3, S4]. Failure to do so results in missing faces at chunk edges.
- **Precision**: Floating-point precision issues may arise when calculating vertex positions if chunk offsets are large. Using integer arithmetic for block coordinates and converting to float only for vertex positions is recommended.
- **Transparent Blocks**: Transparent blocks (e.g., glass) require special handling. They may need to be rendered in a separate pass with depth sorting to avoid z-fighting or incorrect transparency blending [S2, S3].

## Subtopics

### [GPU Voxel Mesh Generation](3670-gpu-voxel-mesh-generation.md)
GPU Voxel Mesh Generation is a technique to generate a static triangle mesh from a 3D voxel grid entirely on the GPU, avoiding CPU readback. It identifies exposed faces of active voxels and generates geometry for each face using a multi-pass compute shader pipeline. Unlike CPU-based meshing, which is significantly slower (e.g., 5 FPS vs >1000 FPS for generation alone), this method leverages parallel processing to achieve high performance. It builds on [Voxel Mesh Generation with Face Culling](2411-voxel-mesh-generation-with-face-culling.md) by implementing the face culling logic within a GPU compute shader environment using indirect drawing.

## Operating characteristics

- **Time Complexity**: $O(N)$, where $N$ is the number of voxels in the chunk. Each voxel is visited once, and for each solid voxel, up to 6 faces are checked.
- **Memory Complexity**: $O(V + I)$, where $V$ is the number of vertices and $I$ is the number of indices in the generated mesh. In the worst case (all voxels exposed), $V \approx 4N$ and $I \approx 6N$.
- **Measured Performance**: CPU-based meshing is significantly slower than GPU-based methods. For example, one source reports 5 FPS for CPU meshing vs >1000 FPS for GPU meshing generation alone [S1, unofficial]. Performance scales with chunk size; larger chunks reduce the overhead of mesh generation per unit volume but increase memory usage.
- **Strengths**: Simple to implement, deterministic, easy to update incrementally (only affected chunks need regeneration), suitable for dynamic worlds with frequent block changes.
- **Limitations**: High vertex count compared to more advanced meshing techniques (e.g., greedy meshing), requires careful handling of chunk boundaries and transparent blocks, CPU-bound for large worlds.
- **When to Use**: Ideal for voxel engines where simplicity and ease of implementation are prioritized, or when the world changes frequently and dynamic mesh updates are required.
- **When Not to Use**: Not suitable for static worlds where maximum performance is critical, or when advanced mesh optimization techniques (e.g., greedy meshing, quadtree-based meshing) are needed.

## Background

Voxel Mesh Generation with Face Culling is a foundational technique in voxel-based game development, originating from early Minecraft clones and sandbox games. It was popularized by the need to render large voxel worlds efficiently on limited hardware. The method is widely used in educational voxel engines and indie games due to its simplicity and effectiveness. Key implementations include [exolands](https://github.com/kvxy/exolands) [S2], [voxel-compare](https://github.com/scarlyle0/voxel-compare) [S3], and [voxel-engine](https://github.com/ernerdgn/voxel-engine) [S4]. The technique has evolved to include optimizations such as baked lighting, texture atlases, and transparent block sorting [S2, S3, S4].

## Sources

[S1] voxel-rs/voxel-rs, https://github.com/voxel-rs/voxel-rs, fetched 2026-10-27 (unofficial)
[S2] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
[S3] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S4] GitHub - ernerdgn/voxel-engine: A voxel engine written from scratch in ..., https://github.com/ernerdgn/voxel-engine, fetched 2026-10-29 (unofficial)
