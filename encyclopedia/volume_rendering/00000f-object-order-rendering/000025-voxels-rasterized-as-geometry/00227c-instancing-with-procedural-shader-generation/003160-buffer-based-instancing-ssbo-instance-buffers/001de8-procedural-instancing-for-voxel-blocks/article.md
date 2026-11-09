# Procedural Instancing for Voxel Blocks

Also called: Voxel Block Procedural Instancing, Procedural Instancing Voxel Rendering, Cubic Grid Voxel Rendering with Instance Buffers

## Overview
Procedural Instancing for Voxel Blocks is a rendering technique that replaces pre-baked mesh geometry with compact instance records, resolving position, color, and occlusion in the vertex shader using a shared cube template [S1, S4, unofficial]. It differs from standard hardware instancing by computing instance transforms and visibility flags procedurally in the vertex stage, avoiding CPU-side mesh generation and allowing custom setup logic for each instance's transformation and parameters [S2, S3, unofficial]. This method significantly reduces memory usage and increases cache capacity compared to baked meshes, though it increases vertex work for zero-gap layouts by submitting whole cubes and discarding hidden faces in the shader [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of occupied voxel positions, material/texture indices, and visibility flags (face masks).
*   **Output:** Rasterized fragments representing the voxel blocks, with correct occlusion and coloring.

**Data Structures:**
The method relies on two primary buffers and a shared geometry template.

1.  **Instance Buffer (SSBO):** Stores per-voxel data. Layouts vary by implementation but generally include position and transformation/visibility data.
    *   *Implementation A (Compact):* 6 bytes per voxel: flat grid index (integer), six-bit surviving-face mask (uint8), orbit byte (uint8) [S1, unofficial].
    *   *Implementation B (Unity StructuredBuffer):* `float4` per instance. Components: `(x, y, z)` for position, `w` for clip parameter [S2, unofficial].
    *   *Implementation C (GLSL std430):* 32 bytes per instance. Fields: `posX` (float, 4B), `posY` (float, 4B), `posZ` (float, 4B), `textureIndex` (int, 4B), `shapeRotation` (uint, 4B), `tintR` (float, 4B), `tintG` (float, 4B), `tintB` (float, 4B) [S4, unofficial]. The `shapeRotation` uint is packed: lower 16 bits are `shape` index, upper 16 bits are `rotation` index [S4, unofficial].
    *   *Implementation D (C# FieldOffset):* 32 bytes. Offset 0: `Vector3` position (12B); Offset 12: `int` texture (4B); Offset 16: `ushort` shape (2B); Offset 18: `ushort` rotation (2B); Offset 20: `Vector3` tint (12B) [S4, unofficial].

2.  **Vertex Buffer:** A shared template geometry. Typically a 24-vertex cube (6 faces × 4 vertices) or a single unit cube [S1, S4, unofficial].
    *   *Layout (GLSL std430):* 32 bytes per vertex. Fields: `position` (vec3, 12B), `_pad0` (float, 4B), `uv` (vec2, 8B), `_pad2` (float, 4B), `_pad1` (float, 4B) [S4, unofficial].

3.  **Uniforms:** View and projection matrices, chunk position offset, and vertex count per shape [S4, unofficial].

**Numbered Steps:**

1.  **Instance Data Preparation:** The CPU or compute shader generates an instance record for each occupied voxel. This record contains the voxel's world position and visibility/texture data [S1, S4, unofficial]. For example, in the compact layout, a 6-byte record is generated per voxel [S1, unofficial]. In the Unity implementation, a `float4` position and clip value are stored in a `StructuredBuffer` [S2, unofficial].

2.  **Draw Call:** A single instanced draw call (e.g., `glDrawArraysInstanced` or Unity's procedural instancing pragma) is issued with an instance count equal to the number of occupied voxels [S1, S2, S4, unofficial].

3.  **Vertex Shader Execution (Per Instance):**
    a.  **Fetch Instance Data:** The vertex shader uses `gl_InstanceID` (or `SV_InstanceID`) to index into the instance buffer [S1, S2, S4, unofficial].
        *   *Unity:* `float4 position = _Positions[unity_InstanceID];` [S2, unofficial].
        *   *GLSL:* `ShapeInstance instance = ShapeInstances[instanceID];` [S4, unofficial].
    b.  **Compute Transform/Visibility:**
        *   *Position:* Construct a translation matrix or vector from the instance position [S2, S3, unofficial]. In GLSL, the local vertex position is adjusted: `vec3 localPos = ShapeVertices[vertexID].position - vec3(0.5);` [S4, unofficial].
        *   *Rotation/Shape:* If using precomputed rotation matrices, retrieve the base rotation matrix from a precomputed array (e.g., `_87`) using the upper 16 bits of the `shapeRotation` field [S4, unofficial]. The rotation matrix represents one of 24 possible orientations [S4, unofficial].
        *   *Transformation:* Apply rotation to the local vertex: `localPos = rotationMatrix * localPos;` [S4, unofficial]. Then translate: `localPos += vec3(instance.posX, instance.posY, instance.posZ);` [S4, unofficial]. Add chunk position if applicable: `localPos += Uniforms.chunkPos;` [S4, unofficial].
        *   *Clip Distance:* Determine visibility. In Unity, `_Clip = -1.0 + isSolid;` where `isSolid` is 1.0 for solid voxels and 0.0 for empty [S3, unofficial]. In other implementations, hidden faces are culled by moving their corners outside the clip volume in the vertex shader [S1, unofficial].
    c.  **Output:**
        *   Transform the final position to clip space: `gl_Position = mul(mul(float4(localPos, 1.0), view), projection);` [S4, unofficial].
        *   Output UV coordinates from the vertex buffer [S4, unofficial].
        *   Output texture index and tint color from the instance buffer [S4, unofficial].

4.  **Fragment Shader Execution:**
    a.  **Clipping:** Discard fragments based on the clip value. In Unity, `clip(_Clip)` is used; if `_Clip < 0`, the fragment is discarded [S3, unofficial]. Since `_Clip` is 0.0 for solid voxels and -1.0 for empty, solid voxels are rendered [S3, unofficial].
    b.  **Texturing:** Sample the texture array using the texture index and UV coordinates [S4, unofficial].
    c.  **Coloring:** Apply tint color and albedo [S2, S4, unofficial].

**Pseudocode (GLSL-style):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
// Vertex Shader
#version 450
layout(location = 0) in vec3 aPos;
layout(location = 1) in vec2 aUV;

layout(std430, binding = 0) buffer InstanceBuffer {
    struct {
        float posX, posY, posZ;
        int textureIndex;
        uint shapeRotation;
        float tintR, tintG, tintB;
    } instances[];
};

layout(std430, binding = 1) buffer VertexBuffer {
    struct {
        vec3 position;
        float _pad0;
        vec2 uv;
        float _pad2;
        float _pad1;
    } vertices[];
};

uniform mat4 view;
uniform mat4 projection;
uniform vec3 chunkPos;
uniform int verticesPerShape;

out vec2 vUV;
out int vTextureIndex;
out vec3 vTint;

void main() {
    uint instanceID = gl_InstanceID;
    uint vertexID = gl_VertexID;
    
    // Fetch instance data
    InstanceBuffer.Instance inst = instances[instanceID];
    
    // Fetch vertex data
    // Calculate index based on shape and rotation
    uint shapeIndex = inst.shapeRotation & 0xFFFFu;
    uint rotationIndex = inst.shapeRotation >> 16u;
    
    // Note: verticesPerShape is the number of vertices per shape instance
    uint vertexIndex = vertexID + shapeIndex * verticesPerShape;
    VertexBuffer.Vertex vert = vertices[vertexIndex];
    
    // Transform vertex
    vec3 localPos = vert.position - vec3(0.5);
    
    // Apply rotation (simplified, assuming rotation matrix lookup)
    // In practice, retrieve rotation matrix from precomputed array using rotationIndex
    // localPos = rotationMatrix * localPos;
    
    localPos += vec3(inst.posX, inst.posY, inst.posZ);
    localPos += chunkPos;
    
    // Output
    gl_Position = projection * view * vec4(localPos, 1.0);
    vUV = vert.uv;
    vTextureIndex = inst.textureIndex;
    vTint = vec3(inst.tintR, inst.tintG, inst.tintB);
}
```

**Parameters:**
*   `verticesPerShape`: Uniform integer specifying the number of vertices per shape instance [S4, unofficial].
*   `chunkPos`: Uniform vec3 for chunk offset [S4, unofficial].
*   `_Color`: Albedo color property (default 1,1,1,1) [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Vertex Work Increase:** Instancing submits whole cubes and discards hidden faces in the vertex shader. For zero-gap layouts, this causes a 2.0× to 3.3× increase in vertex work compared to culling faces on the CPU [S1, unofficial]. This is acceptable as it is vertex work only and no fragments are processed for hidden faces [S1, unofficial].
*   **Precision Drift:** GPU f32 arithmetic drift is bounded to under 1e-7 of the unit box, which is less than half a step of an 8-bit channel, ensuring visual correctness [S1, unofficial].
*   **Matrix Multiplication Order:** Row-major vs column-major matrix multiplication must be handled correctly. A workaround (e.g., `spvWorkaroundRowMajor`) may be needed to ensure correct transformation [S4, unofficial].
*   **Hidden Face Culling:** Hidden faces must be sent outside the clip volume rather than simply not drawn, so culling survives translucent fade-ins [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Geometry build speed is 1.9–4.6× faster than pre-branch core methods [S1, unofficial]. Whole build speed is 1.3–1.8× faster [S1, unofficial].
*   **Memory Complexity:** Memory per voxel is 6 bytes (vs. over 1 kilobyte for six baked quads) [S1, unofficial]. Specimen memory at R=64 is reduced from 42.0 MB to 649 KB [S1, unofficial]. Worker message size at R=64 is reduced from 42.3 MB to 457 KB (92× reduction) [S1, unofficial]. Resident memory for 294 on-screen specimens is reduced from 27.8 MB to 0.66 MB [S1, unofficial].
*   **Performance:** Maintains interactive frame rates of approximately 60 FPS at high voxel densities [S2, unofficial]. Cache capacity at R=64 increases from 6 specimens to about 394 specimens before exhausting `CACHE_BYTES` [S1, unofficial].
*   **Strengths:** Significant memory reduction, increased cache capacity, single draw call, no CPU-side mesh generation.
*   **Limitations:** Increased vertex work for zero-gap layouts (2.0× to 3.3×), requires careful handling of matrix multiplication order and precision.
*   **When to Use:** When memory bandwidth and cache efficiency are critical, and vertex shader performance is sufficient to handle the increased vertex count. Suitable for large voxel scenes with high density.
*   **When Not to Use:** When vertex shader performance is a bottleneck, or when the scene has many zero-gap layouts where face culling is essential.

## Background
This technique is used in projects like the block-showroom by lardratboy [S1, unofficial], and is implemented in Unity using procedural instancing [S2, S3, unofficial]. It is also used in cubic grid voxel renderers [S4, unofficial]. The method evolved from baked mesh rendering to reduce memory overhead and improve performance.

## Sources
[S1] Feature/center instancing, https://github.com/lardratboy/block-showroom/pull/7, fetched 2026-10-26 (unofficial)
[S2] NotABottle/Voxel-Shader, https://github.com/NotABottle/Voxel-Shader, fetched 2026-10-26 (unofficial)
[S3] bzgeb/UnityGPUMeshVoxelizerPart4, https://github.com/bzgeb/UnityGPUMeshVoxelizerPart4, fetched 2026-10-26 (unofficial)
[S4] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
