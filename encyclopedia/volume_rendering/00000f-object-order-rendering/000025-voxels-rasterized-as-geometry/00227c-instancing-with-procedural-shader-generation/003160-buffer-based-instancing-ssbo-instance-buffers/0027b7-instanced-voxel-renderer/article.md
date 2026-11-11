# Instanced Voxel Renderer

Also called: Chunk-based Instanced Voxel Rendering, Cubic-Grid Voxel Renderer, Indirect Rendering via SSBO, SSBO-based Instanced Rendering (Pipeline A)

## Overview
A GPU-driven voxel rendering technique that packs shape instances into large Shader Storage Buffer Objects (SSBOs), groups them into spatial chunks, and renders them via instanced drawing with per-chunk frustum culling [S1, S2, unofficial]. It differs from [Procedural Instancing for Voxel Blocks](1637-procedural-instancing-for-voxel-blocks.md) by using indirect draw calls and a mega-SSBO for face data rather than per-vertex attribute buffers, and from [Cube Rasterization via SSBO](2267-cube-rasterization-via-ssbo.md) by explicitly managing chunk-based spatial indexing and indirect command buffers [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A set of block types (e.g., Grass, Dirt, Stone) defined by their visible faces and associated textures [S1, S2, unofficial].
- A world volume defined by chunks of size `chunkLength` (typically 33 blocks) [S1, unofficial].
- Camera view and projection matrices [S1, unofficial].

**Outputs:**
- Rasterized pixels representing the voxel world [S1, S2, unofficial].

### Data Structures

**1. PackedVertex (SSBO Face Data)**
A 32-bit unsigned integer (`uint32`) packing vertex attributes for a single face corner [S2, unofficial].
- Bits 0-4 (5 bits): X coordinate (relative to chunk/block).
- Bits 5-9 (5 bits): Z coordinate.
- Bits 10-20 (11 bits): Y coordinate.
- Bits 21-23 (3 bits): Face ID (0-5).
- Bits 24-29 (6 bits): Texture Index.
- Bits 30-31: Unused/Reserved.

**2. ShapeInstance (SSBO Instance Data)**
Stored in a condensed SSBO region [S1, unofficial]. Fields include:
- `texture`: Texture atlas index.
- `tint`: Color tint value.
- `shape`: Shape ID.
- `rotation`: Rotation state.
*Note:* Shapes are condensed to a fixed vertex count `verticesPerShape` (max vertices across all shapes), padding missing vertices with a default vertex [S1, unofficial].

**3. DrawIndirectBuffer**
A buffer of `DrawCommand` structs used for indirect rendering [S2, unofficial].
- `count`: Number of vertices to draw (`faceCount * 6`).
- `instanceCount`: Always 1.
- `first`: Offset in the mega-SSBO (`offset * 6`).
- `baseInstance`: Chunk index (used to decode chunk position) [S2, unofficial].

**4. MegaSSBO**
A large `std430` SSBO storing all packed face data. It uses a free-list allocator to manage contiguous blocks of face data [S2, unofficial].

### Steps

1.  **Initialize Texture Array:**
    - Determine `maxX` and `maxY` from all input images [S1, unofficial].
    - Create a `Texture2DArray` with dimensions `maxX`, `maxY`, and `numTextures` [S1, unofficial].
    - Set min/mag filters to `Nearest` and wrap to `Repeat` [S1, unofficial].
    - Upload each image `i` via `GL.TexSubImage3D` at z-offset `i` [S1, unofficial].

2.  **Condense Shapes:**
    - Find `verticesPerShape` = max number of vertices across all input shapes [S1, unofficial].
    - Allocate a flat array of size `numShapes * verticesPerShape` [S1, unofficial].
    - For each shape `s` and vertex `v`, copy `shapeByShapeID[s].Vertices[v]` if `v < length`, else assign a default vertex [S1, unofficial].
    - Upload this array to an SSBO at binding point 3 [S1, unofficial].

3.  **Register Chunks and Allocate SSBO Regions:**
    - For each chunk, calculate `size = shapes.Length * ShapeInstance.MemorySize` [S1, unofficial].
    - If the current SSBO region cannot fit the chunk, allocate a new region [S1, unofficial].
    - Write shape instances to the SSBO at the current `BytePointer` [S1, unofficial].
    - Store `ChunkRenderingData` with `RegionInstanceIndex = index / ShapeInstance.MemorySize` [S1, unofficial].
    - Advance `BytePointer` by `size` [S1, unofficial].
    - *Constraint:* `vramBufferRegionSize` must be `>= chunkVolumeSize` and `<= MaxShaderStorageBlockSize` [S1, unofficial]. If not a multiple of `chunkVolumeSize`, waste `vramBufferRegionSize % chunkVolumeSize` bytes [S1, unofficial].

4.  **Build Indirect Commands:**
    - For each chunk, calculate the number of visible faces `faceCount` [S2, unofficial].
    - Allocate space in the MegaSSBO using a first-fit free-list algorithm [S2, unofficial].
    - Create a `DrawCommand`:
      - `count = faceCount * 6`
      - `instanceCount = 1`
      - `first = offset * 6` (where `offset` is the allocated index in the MegaSSBO)
      - `baseInstance = chunkIndex` [S2, unofficial].
    - Upload commands to the `DrawIndirectBuffer` [S2, unofficial].

5.  **Render Pass:**
    - Compute view/projection matrices and extract frustum planes [S1, unofficial].
    - For each region, bind the corresponding SSBO [S1, unofficial].
    - For each chunk:
      - Compute `offsetPos = chunk.Position - cam.Position` [S1, unofficial].
      - Cull if `!MatrixPlanes.IsBoxInFrustum(planes, offsetPos, offsetPos + Vector3.One * chunkLength)` [S1, unofficial].
      - Set `chunkPos` uniform [S1, unofficial].
      - Call `GL.DrawArraysInstancedBaseInstance(Triangles, 0, verticesPerShape, chunk.Shapes.Length, chunk.RegionInstanceIndex)` [S1, unofficial].

### Vertex Shader Logic
1.  **Decode Chunk Position:**
    - Use `gl_BaseInstance` to decode chunk world position [S2, unofficial].
    - `chunkX = (base & 0xFFFF) - 32768`
    - `chunkZ = ((base >> 16) & 0xFFFF) - 32768` [S2, unofficial].
    - *Note:* Chunk Y position is typically derived from the instance index or a separate uniform, as the base instance primarily encodes X/Z for spatial lookup in this specific implementation [S2, unofficial].

2.  **Unpack Face Data:**
    - Use `gl_VertexID` to determine face and corner index [S2, unofficial].
    - Extract `faceId`, `texIndex`, and coordinates from the packed vertex in the MegaSSBO [S2, unofficial].

3.  **Compute World Position:**
    - `worldPos = chunkWorldPos + blockPos + corner` [S2, unofficial].
    - `blockPos` is `(bx, by, bz)` derived from the instance data [S2, unofficial].
    - `corner` is the offset for the specific corner of the face [S2, unofficial].

4.  **Apply Lighting:**
    - Apply face-specific lighting from a `faceLights` array [S2, unofficial].
    - Typical values: `+X=0.6, -X=0.6, +Y=1.0, -Y=0.3, +Z=0.8, -Z=0.8` [S2, unofficial].

5.  **Texture Coordinates:**
    - Compute UVs using `u_AtlasTileSize = (1.0/atlasWidth, 1.0/atlasHeight)` uniform and `texIndex` [S2, unofficial].

### Parameters
- `chunkLength`: Width/height/depth of a single chunk in blocks. Typical value: 33 [S1, unofficial].
- `vramBufferRegionSize`: VRAM batch size in bytes for SSBO regions. Must be `>= chunkVolumeSize` and `<= MaxShaderStorageBlockSize` [S1, unofficial].
- `faceLights`: Array of lighting multipliers per face [S2, unofficial].
- `u_AtlasTileSize`: Uniform for texture atlas scaling [S2, unofficial].

### Edge Cases and Pitfalls
- **VRAM Limits:** `vramBufferRegionSize` must not exceed hardware `MaxShaderStorageBlockSize` [S1, unofficial].
- **Alignment Waste:** If `vramBufferRegionSize` is not a multiple of `chunkVolumeSize`, it wastes `vramBufferRegionSize % chunkVolumeSize` bytes [S1, unofficial].
- **Dynamic Lookup:** Using `gl_InstanceID` (or `gl_BaseInstance`) for SSBO indexing is dynamic, impacting cache efficiency compared to standard instancing [S1, S2, S4, unofficial].

## Operating characteristics
- **Time Complexity:** O(N) for rendering, where N is the number of visible faces. Frustum culling is performed per-chunk, reducing the number of draw calls and instances processed [S1, unofficial].
- **Memory Complexity:** O(V + F), where V is the number of shape instances and F is the number of visible faces. The MegaSSBO grows with the number of visible faces [S2, unofficial].
- **Strengths:** Efficient culling at the chunk level; flexible shape definitions via condensation; single draw call per chunk reduces CPU overhead [S1, S2, unofficial].
- **Limitations:** Dynamic SSBO lookups in the vertex shader can cause cache misses; VRAM usage scales with visible geometry; chunk size must be carefully chosen to balance culling efficiency and memory waste [S1, S2, unofficial].
- **When to Use:** Large voxel worlds with significant empty space or varying density where chunk-level culling is effective [S1, unofficial].
- **When Not to Use:** Scenes with very high density where almost all chunks are visible, leading to minimal culling benefit but high memory overhead [S1, unofficial].

## Background
The technique is implemented in open-source voxel engines such as `Cubic-Grid_Voxel_Renderer` and `VoxelEngine` [S1, S2, unofficial]. It builds on the principles of buffer-based instancing, specifically leveraging SSBOs for flexible data packing and indirect rendering for scalability [S1, S2, unofficial].

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S2] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
