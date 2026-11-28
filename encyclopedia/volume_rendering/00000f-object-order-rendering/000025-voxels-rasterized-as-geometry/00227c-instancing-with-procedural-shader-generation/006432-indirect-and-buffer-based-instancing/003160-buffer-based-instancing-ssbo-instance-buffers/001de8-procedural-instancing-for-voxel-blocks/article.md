# Procedural Instancing for Voxel Blocks

Also called: Cubic Grid Voxel Rendering with Instance Buffers, Procedural Instancing Voxel Rendering, Voxel Block Procedural Instancing

## Overview
Procedural instancing for voxel blocks replaces baked mesh geometry with a compact instance record per voxel, resolving position, color, and occlusion in the vertex shader using a shared cube template [S1, S2, S3, S4, S5, unofficial]. It differs from standard hardware instancing by computing per-instance transformations and visibility flags dynamically within the vertex shader, allowing custom setup logic without CPU-side mesh generation [S3, S4, unofficial]. This method significantly reduces memory usage compared to baked geometries but increases vertex workload for dense layouts because it submits whole cubes and discards hidden faces in the shader [S1, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A structured buffer (SSBO/StructuredBuffer) containing per-voxel instance data (position, visibility, texture index, rotation) [S1, S3, S5, unofficial]. A shared vertex buffer containing a base geometry (typically a 24-vertex unit cube) [S1, S5, unofficial].
*   **Output:** Rasterized voxels with correct position, orientation, texture, and occlusion culling [S1, S2, S3, S4, S5, unofficial].

**Data Structures:**
The instance data layout varies by implementation but generally includes position and visibility flags [S1, S2, S3, S4, S5, unofficial].
1.  **Compact Record (S1):** 6 bytes per occupied voxel: flat grid index (integer), six-bit surviving-face mask (uint8), orbit byte (uint8) [S1, unofficial].
2.  **Structured Buffer (S3, S4):** `StructuredBuffer<Voxel>` or `StructuredBuffer<float4>` [S2, S3, S4, unofficial]. Fields: `position` (float3 or x,y,z components of float4), `isSolid` (float) or clip parameter (w-component of float4) [S3, S4, unofficial].
3.  **Detailed Instance Struct (S5):** 32 bytes total (GLSL `std430` layout): `posX`, `posY`, `posZ`: `float` (4 bytes each, offset 0, 4, 8) [S5, unofficial]. `textureIndex`: `int` (4 bytes, offset 12) [S5, unofficial]. `shapeRotation`: `uint` (4 bytes, offset 16). Composed of two `ushort`s: lower 16 bits = shape index, upper 16 bits = rotation index [S5, unofficial]. `tintR`, `tintG`, `tintB`: `float` (4 bytes each, offset 20, 24, 28) [S5, unofficial].
4.  **Shared Vertex Buffer (S5):** `ShapeVertex` struct: `position` (vec3, 12 bytes), `_pad0` (float, 4 bytes), `uv` (vec2, 8 bytes), `_pad2` (float, 4 bytes), `_pad1` (float, 4 bytes). Total 32 bytes [S5, unofficial].

**Numbered Steps:**
1.  **Instance Buffer Population:** The application builds a list of occupied voxels [G]. For each voxel, it generates an instance record containing its world position, a solid/empty flag (or face mask), and optional texture/rotation data [S1, S2, S3, S4, S5, unofficial]. This data is uploaded to an SSBO or StructuredBuffer [S1, S3, S5, unofficial].
2.  **Draw Call:** A single instanced draw call is issued (e.g., `glDrawArraysInstanced` or Unity's procedural instancing pragma) with the instance count equal to the number of valid instances in the buffer [S1, S5, unofficial].
3.  **Vertex Shader Lookup:** For each instance, the vertex shader uses the instance ID (`gl_InstanceID` or `unity_InstanceID`) to fetch the instance record from the SSBO [S2, S3, S4, S5, unofficial].
4.  **Transformation Construction:** Construct a transformation matrix `_Matrix` (or `unity_ObjectToWorld`) [S2, S3, S4, unofficial]. Set rotation and scale to identity [S2, S3, S4, unofficial]. Set translation components to the voxel's position coordinates [S2, S3, S4, S5, unofficial]. Some implementations apply additional rotation matrices derived from the `shapeRotation` field if complex orientations are supported [S5, unofficial].
5.  **Vertex Transformation:** Retrieve the base vertex position from the shared vertex buffer [S5, unofficial]. Apply local adjustments (e.g., subtract 0.5 to center the unit cube) [S5, unofficial]. Apply rotation (if any) [S5, unofficial]. Add 0.5 back to center [S5, unofficial]. Add the instance position (translation) [S2, S3, S4, S5, unofficial]. Add any chunk/world offset [S5, unofficial]. Transform the result by the View and Projection matrices [S1, S2, S3, S4, S5, unofficial].
6.  **Visibility Culling (Procedural):** Compute a clip value `_Clip` based on the voxel's solidity [S2, S3, S4, unofficial]. Formula: `_Clip = -1.0 + isSolid` [S2, S4, unofficial]. If `isSolid` is 1.0 (solid voxel), `_Clip` is 0.0 [S2, unofficial]. If `isSolid` is 0.0 (empty voxel), `_Clip` is -1.0 [S2, unofficial]. Alternatively, `_Clip` can be derived from the w-component of the position buffer [S3, unofficial]. In the vertex or fragment shader, use `clip(_Clip)` or move vertices outside the clip volume if `_Clip < 0` [S2, S3, S4, unofficial]. For face-level culling, hidden faces can be pushed outside the clip volume by modifying their vertex positions based on the face mask before transformation [S1, unofficial].
7.  **Output:** Output the transformed position, UV coordinates, texture index, and tint color for the fragment shader [S1, S5, unofficial].

**Pseudocode (Python-like):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def procedural_instancing_voxel_blocks(instance_buffer, shared_vertices, view_matrix, projection_matrix, chunk_position):
    # instance_buffer: list of instance records
    # shared_vertices: list of base vertex records
    # Returns: list of transformed clip positions and varyings
    
    def transform_matrix_vector(m, v):
        # Standard matrix-vector multiplication for 4x4 and 4D vector
        return [
            m[0][0]*v[0] + m[0][1]*v[1] + m[0][2]*v[2] + m[0][3]*v[3],
            m[1][0]*v[0] + m[1][1]*v[1] + m[1][2]*v[2] + m[1][3]*v[3],
            m[2][0]*v[0] + m[2][1]*v[1] + m[2][2]*v[2] + m[2][3]*v[3],
            m[3][0]*v[0] + m[3][1]*v[1] + m[3][2]*v[2] + m[3][3]*v[3]
        ]

    results = []
    for instance_id in range(len(instance_buffer)):
        inst = instance_buffer[instance_id]
        pos = inst['pos']
        is_solid = inst['isSolid']
        tex_idx = inst['texIdx']
        tint = inst['tint']
        
        # Step 6: Compute clip value
        clip_value = -1.0 + is_solid
        
        for vertex_id in range(len(shared_vertices)):
            vert = shared_vertices[vertex_id]
            local_pos = list(vert['pos'])
            
            # Step 5: Vertex Transformation
            # Center adjustment
            local_pos = [x - 0.5 for x in local_pos]
            # Rotation omitted for basic cube; apply if shapeRotation exists
            # local_pos = apply_rotation(local_pos, inst['shapeRotation'])
            local_pos = [x + 0.5 for x in local_pos]
            
            # Translate
            local_pos = [local_pos[i] + pos[i] for i in range(3)]
            # Chunk offset
            local_pos = [local_pos[i] + chunk_position[i] for i in range(3)]
            
            # Transform to clip space
            world_pos = local_pos + [1.0]
            view_pos = transform_matrix_vector(view_matrix, world_pos)
            clip_pos = transform_matrix_vector(projection_matrix, view_pos)
            
            results.append({
                'clip_position': clip_pos,
                'uv': vert['uv'],
                'tex_idx': tex_idx,
                'tint': tint,
                'clip_value': clip_value
            })
    return results
```

**Parameters:**
*   `verticesPerShape`: Uniform integer specifying vertices per shape instance (e.g., 24 for a cube) [S5, unofficial].
*   `_Color`: Albedo color property, default (1,1,1,1) [S4, unofficial].
*   `UNITY_PROCEDURAL_INSTANCING_ENABLED`: Preprocessor define enabling procedural instancing in Unity [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Vertex Work Increase:** Submitting whole cubes and culling in the shader increases vertex work by 2.0× to 3.3× for zero-gap layouts (e.g., solid blocks) compared to only drawing visible faces [S1, unofficial]. This is acceptable as it is vertex-only work and avoids the cost of per-frame buffer rebuilds or multiple draw calls [S1, unofficial].
*   **Precision Drift:** GPU float32 arithmetic drift is bounded to under 1e-7 of the unit box, which is less than half a step of an 8-bit channel, ensuring visual correctness [S1, unofficial].
*   **Matrix Multiplication Order:** Ensure correct row-major vs. column-major handling, especially when porting between languages or using specific shader compilers (e.g., SPIRV_Cross) [S5, unofficial].
*   **Hidden Face Culling:** To survive translucent fade-ins, hidden faces should be moved outside the clip volume rather than simply not drawn [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Geometry build speed is 1.9–4.6× faster than pre-branch core methods; whole build speed is 1.3–1.8× faster [S1, unofficial].
*   **Memory Complexity:** Memory per voxel is reduced to 6 bytes (vs. over 1 kilobyte for six baked quads) [S1, unofficial]. At resolution R=64, specimen memory drops from 42.0 MB to 649 KB, and worker message size drops from 42.3 MB to 457 KB (92× reduction) [S1, unofficial].
*   **Cache Efficiency:** Cache capacity at R=64 increases from 6 specimens to about 394 specimens before exhausting memory limits [S1, unofficial].
*   **Performance:** Maintains interactive frame rates of approximately 60 FPS at high voxel densities [S3, unofficial].
*   **Strengths:** Significant memory reduction, fast build times, single draw call, flexible procedural logic.
*   **Limitations:** Increased vertex workload for dense/zero-gap layouts due to whole-cube submission [S1, unofficial].

## Background
This technique is used in projects like the block-showroom by lardratboy, specifically in `src/scene/instancing.js` and `lattice/recipe.js` [S1, unofficial]. It is also implemented in Unity-based voxel renderers using procedural instancing pragmas [S2, S3, S4, unofficial]. The method leverages procedural instancing features available in modern graphics APIs and engines to avoid CPU-side mesh generation for every voxel [S3, S4, unofficial].

## Sources
[S1] Feature/center instancing, https://github.com/lardratboy/block-showroom/pull/7, fetched 2026-10-26 (unofficial)
[S2] bzgeb/UnityGPUMeshVoxelizerPart5, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart5, fetched 2026-10-26 (unofficial)
[S3] NotABottle/Voxel-Shader, https://github.com/NotABottle/Voxel-Shader, fetched 2026-10-26 (unofficial)
[S4] bzgeb/UnityGPUMeshVoxelizerPart4, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart4, fetched 2026-10-26 (unofficial)
[S5] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
