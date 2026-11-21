# Buffer-Based Instancing (SSBO/Instance Buffers)

Also called: Shader Storage Buffer Object Instancing, GPU-Driven Voxel Instancing

## Overview
Buffer-Based Instancing is a GPU-driven rendering technique that stores per-voxel or per-instance metadata (position, color, shape, rotation) in Shader Storage Buffer Objects (SSBOs) or structured buffers, allowing the vertex shader to fetch and transform geometry procedurally during rasterization [S1, S2, S3, unofficial]. It solves the problem of high vertex buffer overhead and CPU-GPU synchronization latency associated with generating individual meshes for thousands of voxels. It sets itself apart from sibling methods by decoupling geometry definition (static vertex/index buffers) from instance data (dynamic SSBOs), enabling complex, non-uniform shapes and dynamic updates without rebuilding the entire draw call structure [S1, S2, S3, unofficial].

## Shared principles
The core mechanism relies on a single static geometry primitive (e.g., a cube template or point sprite) and a large, GPU-accessible buffer containing instance attributes.

1.  **Data Layout:** Instance data is packed into SSBOs (`std430` in GLSL, `StructuredBuffer` in HLSL) to minimize memory bandwidth. Common packing strategies include:
    *   **Flat Grid Index:** A single integer representing the voxel's position in a 1D array, often combined with a face mask or orbit byte for compactness [S1, unofficial].
    *   **Bit-Packed Coordinates:** Integer coordinates $(x, y, z)$ packed into `uint` or `uvec2` fields using bitmasks (e.g., 11 bits for X, 10 bits for Y, 11 bits for Z) to fit multiple voxels into fewer buffer slots [S2, unofficial].
    *   **Attribute Fields:** Separate fields for texture index, rotation (packed as two 16-bit integers), tint color, and material properties (metalness, roughness) [S3, S2, unofficial].
2.  **Vertex Shader Execution:** The vertex shader receives `gl_VertexID` (local vertex index) and `gl_InstanceID` (instance index). It fetches the corresponding instance record from the SSBO [S1, S2, S3, unofficial].
3.  **Transformation:** The shader computes the world position by combining the instance's base position (from the SSBO) with the local vertex offset. Transformations such as rotation and scaling are applied using matrices or lookup tables stored in uniform buffers or additional SSBOs [S3, S2, unofficial].
4.  **Culling:** Visibility is determined either by hardware clipping (moving hidden vertices outside the clip volume) or by procedural face culling in the vertex shader based on the camera vector [S1, S2, unofficial].

## Subtopics

### [Procedural Instancing for Voxel Blocks](1637-procedural-instancing-for-voxel-blocks.md)
Uses a shared 24-vertex cube template and procedural rotation/translation in the vertex shader. Best for uniform blocks with varied textures.

### [Cube Rasterization via SSBO](2267-cube-rasterization-via-ssbo.md)
Renders explicit geometric cubes using a static index buffer (18 indices for 3 faces) and bit-packed SSBO data. Best for standard voxel lighting with exact geometric culling.

### [Instanced Voxel Renderer](4102-instanced-voxel-renderer.md)
Packs complex, non-uniform shapes into SSBOs with a free-list memory manager and indirect draw calls. Best for large worlds with varied block types and texture atlasing.

## Background
This technique evolved from early hardware instancing (e.g., [WebGL 2 Hardware Instancing](3152-webgl-2-hardware-instancing.md)) by leveraging the high-bandwidth, low-latency access of SSBOs introduced in OpenGL 4.3 and Vulkan. It addresses the scalability limits of CPU-side mesh generation for dynamic voxel worlds, allowing the GPU to drive the rendering pipeline directly from compact data structures [S1, S2, S3, unofficial].

## Sources
