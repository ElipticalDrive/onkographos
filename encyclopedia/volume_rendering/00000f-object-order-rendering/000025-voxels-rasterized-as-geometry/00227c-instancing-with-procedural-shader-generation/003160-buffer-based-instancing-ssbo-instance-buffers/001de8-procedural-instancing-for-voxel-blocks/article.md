# Procedural Instancing for Voxel Blocks

Also called: Cubic Grid Voxel Rendering with Instance Buffers, Feature/center instancing, GalensUnified Cubic Grid Renderer, Procedural Instancing, center instancing, voxel object generator

## Overview
Procedural instancing for voxel blocks replaces baked mesh geometry with a compact instance record per voxel, resolving position, color, and occlusion in the vertex shader using a shared cube template [S1, unofficial]. It separates block data (position, texture, shape, rotation, tint) into an instance buffer and geometry data (vertices) into a vertex buffer, applying transformations in the vertex shader [S3, unofficial]. The technique uses procedural instancing to render voxel blocks as individual instances where position and clip distance are determined by a structured buffer [S2, unofficial]. It builds on [Buffer-Based Instancing (SSBO/Instance Buffers)](6501-buffer-based-instancing-ssbo-instance-buffers.md) by replacing per-voxel mesh generation with a single shared primitive and GPU-driven attribute resolution.

## How it works
**Inputs:** Occupancy data (flat grid index, face mask, orbit byte), texture atlas, shared 24-vertex cube template.
**Outputs:** Rasterized image of the voxel scene.

**Data Structures:**
1. **Instance Buffer:** A GPU-accessible buffer (e.g., `StructuredBuffer` in HLSL or `std430` buffer in GLSL) storing per-instance attributes [S2, S3, unofficial]. The GLSL `std430` layout occupies 32 bytes: `posX`, `posY`, `posZ` (float, 4 bytes each), `textureIndex` (int, 4 bytes), `shapeRotation` (uint, 4 bytes), `tintR`, `tintG`, `tintB` (float, 4 bytes each) [S3, unofficial]. The `shapeRotation` uint packs two ushorts: the lower 16 bits represent the `shape` index, and the upper 16 bits represent the `rotation` index [S3, unofficial]. The Unity implementation uses a `StructuredBuffer<float4>` where the `w` component stores a clip parameter [S2, unofficial]. A compact 6-byte layout exists per occupied voxel: flat grid index (integer), six-bit surviving-face mask (uint8), orbit byte (uint8) [S1, unofficial].
2. **Vertex Buffer:** A static buffer containing a 24-vertex cube template (4 vertices per face × 6 faces) [S1, S3, unofficial]. Each vertex stores position and UV coordinates [S3, unofficial].
3. **Uniforms:** `view` (mat4, 64 bytes), `projection` (mat4, 64 bytes), `chunkPos` (vec3, 12 bytes), `verticesPerShape` (int, 4 bytes) [S3, unofficial].

**Numbered Steps:**
1. Populate the instance buffer with records for each occupied voxel containing position, texture index, shape/rotation, and tint color [S2, S3, unofficial].
2. Issue a single instanced draw call using the shared 24-vertex cube template, with instance count equal to the number of occupied voxels [S1, S3, unofficial].
3. Vertex Shader Execution:
   - Calculate vertex index: `index = gl_VertexID + (shapeRotation & 0xFFFF) * verticesPerShape` [S3, unofficial].
   - Retrieve base rotation matrix: `R = RotationMatrixArray[shapeRotation >> 16]` [S3, unofficial].
   - Transform vertex: `v_centered = v_local - 0.5`, `v_rotated = R * v_centered`, `v_translated = v_rotated + 0.5 + P_instance + P_chunk` [S3, unofficial].
   - Convert to clip space: `v_clip = Projection * View * v_translated` [S3, unofficial]. Apply clip parameter: `v_clip.w = -1.0 + w_component` or move hidden face corners outside the clip volume [S1, S2, unofficial].
   - Output UV coordinates, texture index, and tint color [S3, unofficial].
4. Fragment Shader Execution: Sample the texture atlas using interpolated UV coordinates and texture index, then apply the tint color to the sampled albedo [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vertex_shader(gl_VertexID, gl_InstanceID):
    # Retrieve instance data
    instance = instance_buffer[gl_InstanceID]
    shape_idx = instance.shapeRotation & 0xFFFF
    rot_idx = instance.shapeRotation >> 16
    
    # Retrieve base vertex
    v_local = vertex_buffer[gl_VertexID + shape_idx * verticesPerShape]
    
    # Apply rotation and translation
    R = rotation_matrix_array[rot_idx]
    v_centered = v_local.position - 0.5
    v_rotated = R @ v_centered
    v_translated = v_rotated + 0.5 + instance.position + chunkPos
    
    # Convert to clip space
    v_clip = projection @ (view @ vec4(v_translated, 1.0))
    
    # Handle clipping/occlusion
    if instance.clip_param is not None:
        v_clip.w = -1.0 + instance.clip_param
        
    return v_clip, v_local.uv, instance.textureIndex, instance.tint
```

**Formulas:**
- Vertex Index: $v_{index} = gl\_VertexID + (shapeRotation \ \& \ 0xFFFF) \times verticesPerShape$ [S3, unofficial].
- Rotation Matrix Lookup: $R = \text{RotationMatrixArray}[shapeRotation \gg 16]$ [S3, unofficial].
- Position Transformation: $v_{centered} = v_{local} - 0.5$, $v_{rotated} = R \times v_{centered}$, $v_{translated} = v_{rotated} + 0.5 + P_{instance} + P_{chunk}$ [S3, unofficial].
- Clip Space Conversion: $v_{clip} = Projection \times View \times v_{translated}$ [S3, unofficial].
- Clip Distance Adjustment: $v_{clip}.w = -1.0 + w_{component}$ [S2, unofficial].

**Parameters:**
- `verticesPerShape`: Uniform integer specifying the number of vertices per shape (typically 24 for a cube) [S3, unofficial].
- `chunkPos`: Uniform `vec3` for chunk-based positioning [S3, unofficial].
- `view`, `projection`: Uniform matrices for camera transformation [S3, unofficial].

**Edge Cases and Pitfalls:**
- Hidden faces are submitted as whole cubes and culled by moving corners outside the clip volume in the vertex shader, increasing vertex work by 2.0× to 3.3× for zero-gap layouts [S1, unofficial].
- GPU float32 arithmetic drift is bounded to under $10^{-7}$ of the unit box, which is less than half a step of an 8-bit channel, ensuring visual correctness [S1, unofficial].
- Row-major vs column-major matrix storage requires a workaround (e.g., `spvWorkaroundRowMajor`) to ensure correct transformation [S3, unofficial].
- Axis center tables must remain `Float64Array` on the CPU to prevent precision drift, converting to `f32` only for GPU copy [S1, unofficial].

## Operating characteristics
- **Memory Usage:** 6 bytes per voxel versus over 1 kilobyte for six baked quads [S1, unofficial]. At resolution R=64, specimen memory reduces from 42.0 MB to 649 KB [S1, unofficial].
- **Performance:** Geometry build speed is 1.9–4.6× faster than pre-branch core methods, and whole build speed is 1.3–1.8× faster [S1, unofficial]. Maintains interactive frame rates of approximately 60 FPS at high voxel densities [S2, unofficial].
- **Cache Efficiency:** Cache capacity increases from 6 specimens to about 394 specimens at R=64 before exhausting `CACHE_BYTES` [S1, unofficial].
- **Limitations:** Increased vertex work for dense, zero-gap layouts due to submitting whole cubes and culling in the shader [S1, unofficial].

## Background
This technique is used in the block-showroom project by lardratboy [S1, unofficial] and the GalensUnified Cubic Grid Renderer [S3, unofficial]. It leverages GPU instancing capabilities to render large numbers of voxel blocks efficiently by minimizing data transfer and draw calls [S3, unofficial]. The method separates block data into an instance buffer and geometry data into a vertex buffer, applying transformations in the vertex shader [S3, unofficial].

## Sources
[S1] Feature/center instancing, https://github.com/lardratboy/block-showroom/pull/7, fetched 2026-10-26 (unofficial)
[S2] NotABottle/Voxel-Shader, https://github.com/NotABottle/Voxel-Shader, fetched 2026-10-26 (unofficial)
[S3] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
