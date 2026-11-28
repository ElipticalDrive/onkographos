# Instanced Voxel Renderer

Also called: Chunk-based Instanced Voxel Rendering, Cubic-Grid Voxel Renderer, Indirect Rendering via SSBO, SSBO-based Instanced Rendering (Pipeline A)

## Overview
The Instanced Voxel Renderer is a GPU-driven technique that packs shape instances into large Shader Storage Buffer Objects (SSBOs), groups them into spatial chunks, and renders them via instanced drawing with per-chunk frustum culling [S1, unofficial][S2, unofficial]. It builds on [Buffer-Based Instancing (SSBO/Instance Buffers)](6501-buffer-based-instancing-ssbo-instance-buffers.md) by reconstructing geometry procedurally in the vertex shader from packed bit-fields rather than relying on standard vertex attributes [S1, unofficial][S2, unofficial]. Unlike [Procedural Instancing for Voxel Blocks](1637-procedural-instancing-for-voxel-blocks.md), this method condenses irregular shapes into fixed-size arrays and uses indirect draw calls to manage variable face counts per chunk [S1, unofficial][S2, unofficial].

## How it works
The method operates in two phases: data preparation (CPU-side or compute-shader driven) and rendering (GPU-side) [S1, unofficial][S2, unofficial].

### Inputs and Outputs
**Inputs:**
- A set of block types, each defined by a list of vertices and face indices [S1, unofficial].
- A texture atlas (2D array) containing textures for block faces [S1, unofficial].
- World data structured as chunks of fixed size ($L \times L \times L$ blocks) [S1, unofficial].

**Outputs:**
- A single draw call (or indirect draw call array) rendering all visible faces [S1, unofficial][S2, unofficial].
- A rendered image with lighting applied per face [S2, unofficial].

### Data Structures
#### 1. Packed Vertex (SSBO Layout)
Each vertex in the SSBO is packed into a single 32-bit unsigned integer (`uint32`) to minimize memory bandwidth [S2, unofficial]. The bit layout is:
- Bits 0–4 (5 bits): $x$ coordinate (relative to chunk) [S2, unofficial].
- Bits 5–9 (5 bits): $z$ coordinate (relative to chunk) [S2, unofficial].
- Bits 10–20 (11 bits): $y$ coordinate (relative to chunk) [S2, unofficial].
- Bits 21–23 (3 bits): Face ID (0–7, representing the 6 faces + 2 dummy/unused) [S2, unofficial].
- Bits 24–29 (6 bits): Texture Index (into the texture atlas) [S2, unofficial].
- Bits 30–31: Unused/Reserved [S2, unofficial].

#### 2. ShapeInstance (SSBO Record)
Used to store per-instance metadata [S1, unofficial]. Fields include:
- `texture`: Index into the texture atlas [S1, unofficial].
- `tint`: Color multiplier [S1, unofficial].
- `shape`: Reference to the base shape definition [S1, unofficial].
- `rotation`: Orientation data [S1, unofficial].
The memory size is fixed per instance (`ShapeInstance.MemorySize`) to allow flat array allocation [S1, unofficial].

#### 3. Condensed Shapes Buffer
To handle shapes with varying vertex counts, a "condensed" flat array is created [S1, unofficial]:
- Size: $N_{shapes} \times V_{max}$, where $V_{max}$ is the maximum vertex count across all input shapes [S1, unofficial].
- Layout: For each shape $s$, vertices $v < V_{max}$ are copied; remaining slots are padded with a default vertex [S1, unofficial].
- This buffer is uploaded to an SSBO at a specific binding point [S1, unofficial].

#### 4. DrawIndirectBuffer
Stores indirect draw commands for the GPU [S2, unofficial]:
- `count`: Number of vertices to draw ($faceCount \times 6$) [S2, unofficial].
- `instanceCount`: 1 [S2, unofficial].
- `first`: Offset in the mega-SSBO (in vertices) $\times 6$ [S2, unofficial].
- `baseInstance`: The chunk index, used to decode chunk position in the vertex shader [S2, unofficial].

### Numbered Steps
1. **Initialize Texture Atlas:** Determine maximum width ($W$) and height ($H$) from all input textures [S1, unofficial]. Create a `Texture2DArray` with dimensions $(W, H, N_{textures})$ [S1, unofficial]. Set min/mag filters to `Nearest`, wrap mode to `Repeat` [S1, unofficial]. Upload each texture $i$ using `GL.TexSubImage3D` at z-offset $i$ [S1, unofficial].
2. **Condense Shapes (CPU/Compute):** Find $V_{max} = \max_{s}(\text{vertices}(s).\text{length})$ [S1, unofficial]. Allocate `condensedShapes` array of size $N_{shapes} \times V_{max}$ [S1, unofficial]. For each shape $s$ and vertex index $v$: if $v < \text{vertices}(s).\text{length}$, copy vertex data; else, assign a default vertex (e.g., zero position) [S1, unofficial]. Upload to SSBO binding point 3 [S1, unofficial].
3. **Register Chunks (CPU/Compute):** For each chunk, calculate required SSBO space: $\text{size} = \text{shapes.Length} \times \text{ShapeInstance.MemorySize}$ [S1, unofficial]. If the current SSBO region cannot fit the chunk, allocate a new region (subject to `vramBufferRegionSize` constraints) [S1, unofficial]. Write shape instances to the SSBO at the current `BytePointer` [S1, unofficial]. Store `ChunkRenderingData` including `RegionInstanceIndex` = index / `ShapeInstance.MemorySize` [S1, unofficial]. Advance `BytePointer` by `size` [S1, unofficial].
4. **Vertex Shader Execution (Per Vertex):** Unpack face data from the SSBO using `gl_VertexID` to determine the face and corner index [S2, unofficial]. Decode chunk position from `gl_BaseInstance` (or `baseInstance` in indirect draw): $\text{chunkX} = (\text{base} \ \& \ 0xFFFF) - 32768$, $\text{chunkZ} = ((\text{base} \gg 16) \ \& \ 0xFFFF) - 32768$ [S2, unofficial]. Compute world position: $\text{WorldPos} = \text{ChunkWorldPos} + \text{BlockPos} + \text{Corner}$, where $\text{BlockPos} = (bx, by, bz)$ and $\text{Corner}$ is the corner offset from `faceCorners[faceId][cornerIndex]` [S2, unofficial]. Apply face-specific lighting from `faceLights` array [S2, unofficial].
5. **Frustum Culling (CPU):** Extract frustum planes from the view/projection matrices [S1, unofficial]. For each chunk, compute offset vector: $\text{offsetPos} = \text{chunk.Position} - \text{cam.Position}$ [S1, unofficial]. Cull chunk if `!MatrixPlanes.IsBoxInFrustum(planes, offsetPos, offsetPos + Vector3.One * chunkLength)` [S1, unofficial].
6. **Rendering:** Bind the SSBO. Set `chunkPos` uniform [S1, unofficial]. Issue draw call using the indirect buffer or direct instanced call [S1, unofficial][S2, unofficial].

### Formulas
- Chunk position decoding: $\text{chunkX} = (\text{base} \ \& \ 0xFFFF) - 32768$, $\text{chunkZ} = ((\text{base} \gg 16) \ \& \ 0xFFFF) - 32768$ [S2, unofficial].
- World position: $\text{WorldPos} = \text{ChunkWorldPos} + \text{BlockPos} + \text{Corner}$ [S2, unofficial].
- Frustum culling bounding box: $\text{offsetPos} = \text{chunk.Position} - \text{cam.Position}$, box spans $\text{offsetPos}$ to $\text{offsetPos} + \text{Vector3.One} \times \text{chunkLength}$ [S1, unofficial].

### Parameters
- **chunkLength**: Width/height/depth of a single chunk in blocks. Typical value: 33 [S1, unofficial].
- **vramBufferRegionSize**: VRAM batch size in bytes for SSBO regions. Must be $\ge$ `chunkVolumeSize` and $\le$ `MaxShaderStorageBlockSize` [S1, unofficial].
- **Face Lights**: Default lighting coefficients: +X=0.6, -X=0.6, +Y=1.0, -Y=0.3, +Z=0.8, -Z=0.8 [S2, unofficial].
- **Atlas Tile Size**: Passed as uniform `u_AtlasTileSize = (1.0/atlasWidth, 1.0/atlasHeight)` [S2, unofficial].

### Edge Cases and Pitfalls
- **VRAM Limits**: `vramBufferRegionSize` must not exceed the hardware `MaxShaderStorageBlockSize`. If it does, an exception must be thrown [S1, unofficial].
- **Chunk Size Constraint**: `vramBufferRegionSize` must be $\ge$ `chunkVolumeSize`. If less, an exception must be thrown [S1, unofficial].
- **Memory Waste**: If `vramBufferRegionSize` is not a multiple of `chunkVolumeSize`, it wastes `waste = vramBufferRegionSize % chunkVolumeSize` bytes [S1, unofficial].
- **Bit-Field Overflow**: The packed vertex format uses 5 bits for X and Z, covering 0–31. If `chunkLength` exceeds 32, coordinates must be offset or the packing scheme must be adjusted [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_instanced_voxel_renderer(chunks, shapes, textures, camera, projection):
    # 1. Initialize Texture Atlas
    atlas_width = max(t.width for t in textures)
    atlas_height = max(t.height for t in textures)
    texture_array = create_texture_2d_array(atlas_width, atlas_height, len(textures))
    texture_array.min_filter = NEAREST
    texture_array.wrap_mode = REPEAT
    for i, tex in enumerate(textures):
        texture_array.sub_image_3d(tex, z_offset=i)

    # 2. Condense Shapes
    vertices_per_shape = max(len(s.vertices) for s in shapes)
    condensed_shapes = allocate_flat_array(len(shapes) * vertices_per_shape)
    for s_idx, shape in enumerate(shapes):
        for v_idx in range(vertices_per_shape):
            if v_idx < len(shape.vertices):
                condensed_shapes[s_idx * vertices_per_shape + v_idx] = shape.vertices[v_idx]
            else:
                condensed_shapes[s_idx * vertices_per_shape + v_idx] = default_vertex()
    bind_ssbo(binding=3, buffer=condensed_shapes)

    # 3. Register Chunks & Build Indirect Commands
    mega_ssbo = allocate_ssbo(vram_buffer_region_size)
    indirect_commands = []
    byte_ptr = 0
    for chunk_idx, chunk in enumerate(chunks):
        offset_pos = chunk.position - camera.position
        if not is_box_in_frustum(offset_pos, offset_pos + vector3(1, 1, 1) * chunk_length):
            continue
        chunk_size = len(chunk.shapes) * ShapeInstance.memory_size
        if byte_ptr + chunk_size > mega_ssbo.capacity:
            mega_ssbo.allocate_new_region()
        mega_ssbo.write(byte_ptr, chunk.shapes)
        region_instance_index = byte_ptr // ShapeInstance.memory_size
        byte_ptr += chunk_size

        face_count = count_chunk_faces(chunk)
        indirect_commands.append(DrawCommand(
            count=face_count * 6,
            instance_count=1,
            first=byte_ptr // 6,
            base_instance=chunk_idx
        ))

    # 4. Rendering
    bind_ssbo(binding=0, buffer=mega_ssbo)
    set_uniform("chunk_pos", camera.position)
    set_uniform("u_atlas_tile_size", (1.0 / atlas_width, 1.0 / atlas_height))
    bind_indirect_buffer(indirect_commands)
    draw_arrays_indirect(mode=TRIANGLES, first=0)
```

## Subtopics

### [Per-Frame SSBO Reconstruction for Voxel Buckets](16072-per-frame-ssbo-reconstruction-for-voxel-buckets.md)
A variant that discards and recreates SSBOs every frame per bucket, simplifying synchronization at the cost of bandwidth.

## Operating characteristics
- **Time Complexity**: CPU-side culling is $O(N_{chunks})$ [S1, unofficial]. Rendering is $O(N_{visible\_faces})$ via indirect draw calls [S2, unofficial].
- **Memory Complexity**: SSBO size scales with the number of chunks and their face counts [S1, unofficial][S2, unofficial]. The condensed shapes buffer scales with $N_{shapes} \times V_{max}$ [S1, unofficial].
- **Performance**: High throughput due to reduced draw calls and efficient SSBO packing [S1, unofficial][S2, unofficial]. However, dynamic SSBO indexing in the vertex shader can impact cache efficiency compared to standard instancing [G].
- **Strengths**: Flexible shape definitions, efficient culling, reduced driver overhead [S1, unofficial][S2, unofficial].
- **Limitations**: Complex data preparation, potential VRAM fragmentation, bit-field precision limits [S1, unofficial][S2, unofficial].

## Background
The Instanced Voxel Renderer is an implementation of GPU-driven rendering techniques applied to voxel engines [S1, unofficial][S2, unofficial]. It draws from general practices in procedural geometry generation and indirect rendering [S1, unofficial][S2, unofficial].

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S2] adrienlabayle/VoxelEngine, https://github.com/adrienlabayle/VoxelEngine, fetched 2026-10-27 (unofficial)
