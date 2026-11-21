# Instanced Voxel Renderer

Also called: Chunk-based Instanced Voxel Rendering, Cubic-Grid Voxel Renderer, Indirect Rendering via SSBO, SSBO-based Instanced Rendering (Pipeline A)

## Overview
The Instanced Voxel Renderer is a GPU-driven technique that packs shape instances into large Shader Storage Buffer Objects (SSBOs), groups them into spatial chunks, and renders them via instanced drawing with per-chunk frustum culling [S1, S2, unofficial]. It builds on [Buffer-Based Instancing (SSBO/Instance Buffers)](6501-buffer-based-instancing-ssbo-instance-buffers.md) by introducing a "mega-SSBO" architecture where geometry is reconstructed procedurally in the vertex shader from packed bit-fields, rather than relying on standard vertex attributes [S1, S2, unofficial]. Unlike [Procedural Instancing for Voxel Blocks](1637-procedural-instancing-for-voxel-blocks.md) which may use simpler per-vertex data, this method condenses irregular shapes into fixed-size arrays and uses indirect draw calls to manage variable face counts per chunk [S1, S2, unofficial].

## How it works
The method operates in two phases: data preparation (CPU-side or compute-shader driven) and rendering (GPU-side).

### Inputs and Outputs
**Inputs:**
- A set of block types, each defined by a list of vertices and face indices.
- A texture atlas (2D array) containing textures for block faces.
- World data structured as chunks of fixed size ($L \times L \times L$ blocks).

**Outputs:**
- A single draw call (or indirect draw call array) rendering all visible faces.
- A rendered image with lighting applied per face.

### Data Structures

#### 1. Packed Vertex (SSBO Layout)
Each vertex in the SSBO is packed into a single 32-bit unsigned integer (`uint32`) to minimize memory bandwidth [S2, unofficial]. The bit layout is:
- Bits 0–4 (5 bits): $x$ coordinate (relative to chunk).
- Bits 5–9 (5 bits): $z$ coordinate (relative to chunk).
- Bits 10–20 (11 bits): $y$ coordinate (relative to chunk).
- Bits 21–23 (3 bits): Face ID (0–7, representing the 6 faces + 2 dummy/unused).
- Bits 24–29 (6 bits): Texture Index (into the texture atlas).
- Bits 30–31: Unused/Reserved.

*Note: The sources specify 5 bits for X and Z, allowing values 0–31. With a typical chunk length of 33 [S1, unofficial], this implies coordinates are offset or the chunk size is effectively limited to 32 for packing purposes, or the implementation uses a specific offset scheme not fully detailed in the bit-field definition alone. The source [S2, unofficial] explicitly defines this layout.*

#### 2. ShapeInstance (SSBO Record)
Used in the [S1, unofficial] implementation to store per-instance metadata. Fields include:
- `texture`: Index into the texture atlas.
- `tint`: Color multiplier.
- `shape`: Reference to the base shape definition.
- `rotation`: Orientation data.
The memory size is fixed per instance (`ShapeInstance.MemorySize`) to allow flat array allocation [S1, unofficial].

#### 3. Condensed Shapes Buffer
To handle shapes with varying vertex counts, a "condensed" flat array is created [S1, unofficial]:
- Size: $N_{shapes} \times V_{max}$, where $V_{max}$ is the maximum vertex count across all input shapes.
- Layout: For each shape $s$, vertices $v < V_{max}$ are copied; remaining slots are padded with a default vertex.
- This buffer is uploaded to an SSBO at a specific binding point [S1, unofficial].

#### 4. DrawIndirectBuffer
Stores indirect draw commands for the GPU [S2, unofficial]:
- `count`: Number of vertices to draw ($faceCount \times 6$).
- `instanceCount`: 1 (since each chunk is drawn as a single instanced batch of faces, or rather, the instance ID selects the chunk).
- `first`: Offset in the mega-SSBO (in vertices) $\times 6$.
- `baseInstance`: The chunk index, used to decode chunk position in the vertex shader.

### Numbered Steps

1. **Initialize Texture Atlas:**
   - Determine maximum width ($W$) and height ($H$) from all input textures.
   - Create a `Texture2DArray` with dimensions $(W, H, N_{textures})$.
   - Set min/mag filters to `Nearest`, wrap mode to `Repeat`.
   - Upload each texture $i$ using `GL.TexSubImage3D` at z-offset $i$ [S1, unofficial].

2. **Condense Shapes (CPU/Compute):**
   - Find $V_{max} = \max_{s}(\text{vertices}(s).\text{length})$.
   - Allocate `condensedShapes` array of size $N_{shapes} \times V_{max}$.
   - For each shape $s$ and vertex index $v$:
     - If $v < \text{vertices}(s).\text{length}$, copy vertex data.
     - Else, assign a default vertex (e.g., zero position).
   - Upload to SSBO binding point 3 [S1, unofficial].

3. **Register Chunks (CPU/Compute):**
   - For each chunk, calculate required SSBO space: $\text{size} = \text{shapes.Length} \times \text{ShapeInstance.MemorySize}$ [S1, unofficial].
   - If the current SSBO region cannot fit the chunk, allocate a new region (subject to `vramBufferRegionSize` constraints) [S1, unofficial].
   - Write shape instances to the SSBO at the current `BytePointer`.
   - Store `ChunkRenderingData` including `RegionInstanceIndex` = index / `ShapeInstance.MemorySize` [S1, unofficial].
   - Advance `BytePointer` by `size` [S1, unofficial].

4. **Vertex Shader Execution (Per Vertex):**
   - Unpack face data from the SSBO using `gl_VertexID` to determine the face and corner index [S2, unofficial].
   - Decode chunk position from `gl_BaseInstance` (or `baseInstance` in indirect draw):
     - $\text{chunkX} = (\text{base} \ \& \ 0xFFFF) - 32768$
     - $\text{chunkZ} = ((\text{base} \gg 16) \ \& \ 0xFFFF) - 32768$
     - *Note: The Y coordinate of the chunk is typically derived from the instance index or a separate uniform, as the 32-bit base instance only encodes X and Z in this specific bit-decoding scheme [S2, unofficial].*
   - Compute world position:
     $$ \text{WorldPos} = \text{ChunkWorldPos} + \text{BlockPos} + \text{Corner} $$
     Where $\text{BlockPos} = (bx, by, bz)$ and $\text{Corner}$ is the corner offset from `faceCorners[faceId][cornerIndex]` [S2, unofficial].
   - Apply face-specific lighting from `faceLights` array [S2, unofficial].

5. **Frustum Culling (CPU):**
   - Extract frustum planes from the view/projection matrices [S1, unofficial].
   - For each chunk, compute offset vector: $\text{offsetPos} = \text{chunk.Position} - \text{cam.Position}$ [S1, unofficial].
   - Cull chunk if `!MatrixPlanes.IsBoxInFrustum(planes, offsetPos, offsetPos + Vector3.One * chunkLength)` [S1, unofficial].

6. **Rendering:**
   - Bind the SSBO.
   - Set `chunkPos` uniform [S1, unofficial].
   - Issue draw call:
     ```python
     GL.DrawArraysInstancedBaseInstance(
         PrimitiveType.Triangles,
         0,
         verticesPerShape,
         chunk.Shapes.Length,
         chunk.RegionInstanceIndex
     )
     ```
     *Note: [S1, unofficial] uses `DrawArraysInstancedBaseInstance` with `verticesPerShape` as the vertex count, implying the geometry is pre-packed into a fixed-size buffer per shape instance. [S2, unofficial] uses indirect drawing with variable face counts. The method supports both variants depending on the specific implementation (condensed vs. packed faces).* [S1, S2, unofficial].

### Parameters
- **chunkLength**: Width/height/depth of a single chunk in blocks. Typical value: 33 [S1, unofficial].
- **vramBufferRegionSize**: VRAM batch size in bytes for SSBO regions. Must be $\ge$ `chunkVolumeSize` and $\le$ `MaxShaderStorageBlockSize` [S1, unofficial].
- **Face Lights**: Default lighting coefficients: +X=0.6, -X=0.6, +Y=1.0, -Y=0.3, +Z=0.8, -Z=0.8 [S2, unofficial].
- **Atlas Tile Size**: Passed as uniform `u_AtlasTileSize = (1.0/atlasWidth, 1.0/atlasHeight)` [S2, unofficial].

### Edge Cases and Pitfalls
- **VRAM Limits**: `vramBufferRegionSize` must not exceed the hardware `MaxShaderStorageBlockSize`. If it does, an exception must be thrown [S1, unofficial].
- **Chunk Size Constraint**: `vramBufferRegionSize` must be $\ge$ `chunkVolumeSize`. If less, an exception must be thrown [S1, unofficial].
- **Memory Waste**: If `vramBufferRegionSize` is not a multiple of `chunkVolumeSize`, it wastes `waste = vramBufferRegionSize % chunkVolumeSize` bytes [S1, unofficial].
- **Bit-Field Overflow**: The packed vertex format uses 5 bits for X and Z. If `chunkLength` > 32, coordinates must be offset or the packing scheme must be adjusted, as 5 bits only cover 0–31 [S2, unofficial]. The source [S1, unofficial] uses a chunk length of 33, which may imply a different coordinate system or offset handling not explicitly detailed in the bit-field definition [S1, S2, unofficial].

## Subtopics

### [Per-Frame SSBO Reconstruction for Voxel Buckets](16072-per-frame-ssbo-reconstruction-for-voxel-buckets.md)
A variant that discards and recreates SSBOs every frame per bucket, simplifying synchronization at the cost of bandwidth.

## Operating characteristics
- **Time Complexity**: CPU-side culling is $O(N_{chunks})$. Rendering is $O(N_{visible\_faces})$ via indirect draw calls.
- **Memory Complexity**: SSBO size scales with the number of chunks and their face counts. The condensed shapes buffer scales with $N_{shapes} \times V_{max}$.
- **Performance**: High throughput due to reduced draw calls and efficient SSBO packing. However, dynamic SSBO indexing in the vertex shader can impact cache efficiency compared to standard instancing [G].
- **Strengths**: Flexible shape definitions, efficient culling, reduced driver overhead.
- **Limitations**: Complex data preparation, potential VRAM fragmentation, bit-field precision limits.

## Background
The Instanced Voxel Renderer is an implementation of GPU-driven rendering techniques applied to voxel engines. It draws from general practices in procedural geometry generation and indirect rendering [S1, S2, unofficial]. The specific implementations cited are from community-driven voxel engine projects [S1, S2, unofficial].

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S2] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
