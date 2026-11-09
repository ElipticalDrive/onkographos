# Buffer-Based Instancing (SSBO/Instance Buffers)

Also called: SSBO Instancing, Shader Storage Buffer Instancing, Procedural Instance Lookup

## Overview
Buffer-Based Instancing is a rendering technique that replaces standard hardware instance attribute buffers with Shader Storage Buffer Objects (SSBOs) or Structured Buffers, allowing the vertex shader to procedurally look up per-instance data (such as position, rotation, and material properties) using the `gl_InstanceID` or a derived index [S1, S2, S3, 11005]. It solves the problem of memory bandwidth and driver overhead associated with binding large, rigid instance buffers by enabling dense, bit-packed, or dynamically allocated data layouts that are accessed entirely within the shader pipeline [S1, S3, 11005]. It differs from standard hardware instancing, which passes per-instance attributes via a second vertex buffer with fixed strides, by performing the data lookup procedurally in the shader, which allows for complex, variable-size instance records and dynamic memory management but incurs higher latency due to the lack of hardware precaching [11005].

## Shared principles
The core mechanism relies on the separation of geometry definition from instance state. The geometry (e.g., a unit cube) is defined in a static Vertex Buffer Object (VBO) or generated procedurally. The instance state is stored in a Shader Storage Buffer Object (SSBO) bound at a specific binding point.

1.  **Instance Identification:** The vertex shader receives the instance identifier. In standard OpenGL, this is `gl_InstanceID`. In indirect or complex indexing schemes, it may be derived from `gl_VertexID` or passed as a semantic value like `SV_InstanceID` [S2, 11005].
2.  **Data Lookup:** The shader uses the instance ID to index into the SSBO. The SSBO layout is defined by the application (e.g., `std430` in GLSL or `cbuffer`/`StructuredBuffer` in HLSL) [S1, S3, 11005].
3.  **Transformation:** The fetched data (position, rotation, scale, material) is used to transform the base geometry vertices into world and clip space [S1, S2, S3].
4.  **Culling/Visibility:** Visibility logic is often implemented in the vertex shader by modifying clip-space coordinates (e.g., pushing vertices to infinity) or by discarding fragments, based on the fetched instance data [S1, S2].

## Subtopics

### [Procedural Instancing for Voxel Blocks](1637-procedural-instancing-for-voxel-blocks.md)
Procedural Instancing for Voxel Blocks replaces baked mesh geometry with a compact instance record per voxel, resolving position, color, and occlusion in the vertex shader using a shared cube template. It differs from standard instancing by computing transformations, visibility culling, and texture selection procedurally within the shader based on a dense, bit-packed instance buffer, allowing for significant memory reduction and cache efficiency compared to pre-baked geometries.

### [Cube Rasterization via SSBO](2267-cube-rasterization-via-ssbo.md)
Cube Rasterization via SSBO renders a set of voxels as explicit geometric cubes by issuing a single `glDrawElementsInstanced` call, where each instance is transformed and colored based on data fetched from a Shader Storage Buffer Object (SSBO). It differs from point-sprite or ray-marched voxel techniques by rasterizing actual triangle geometry (specifically 3 faces per voxel). This approach provides exact cube geometry and explicit face culling logic, at the cost of increased memory bandwidth compared to point-based methods.

### [Instanced Voxel Renderer](4102-instanced-voxel-renderer.md)
The Instanced Voxel Renderer is a GPU-driven technique that packs voxel shape instances into large Shader Storage Buffer Objects (SSBOs), groups them into spatial chunks, and renders them via instanced drawing with per-chunk frustum culling. It builds on Buffer-Based Instancing by introducing chunk-based spatial partitioning and indirect draw calls to minimize CPU-GPU synchronization overhead. Unlike Procedural Instancing for Voxel Blocks which generates geometry procedurally per-vertex, or Cube Rasterization via SSBO which may use simpler flat indexing, this method explicitly manages VRAM regions for chunks and uses a free-list allocator within the SSBO to handle variable face counts per chunk.

### [Hardware Instancing (Instance ID Lookup)](11005-hardware-instancing-instance-id-lookup.md)
Hardware Instancing (Instance ID Lookup) is a rendering technique that draws multiple instances of a geometry using a single draw call by utilizing a system-provided Instance ID to look up instance-specific data (such as transformation matrices) from a buffer or constant buffer within the vertex shader. It differs from standard hardware instancing, which uses a second vertex buffer bound to the draw context to pass per-instance attributes directly to the vertex shader, by performing the data lookup procedurally in the shader. This approach is less efficient than the standard method because the driver cannot inline the data, and hardware precaching on the GPU wavefront is wasted as the hardware must look up the related array entry for each vertex.

### Render to Vertex Buffer (R2VB) with PBO
#### definition
- A technique using Vertex Buffer Objects (VBO) and the Render to Vertex Buffer (R2VB) extension, combined with Pixel Buffer Object (PBO) to load 2D texture values into VBOs, keeping data flow entirely on the GPU [S1, unofficial].
#### usage
- Used in Buckets Texture construction and for the rendering stage [S1, unofficial].

## Background
Buffer-Based Instancing evolved as a solution to the limitations of standard hardware instancing when dealing with highly dynamic or complex instance data. Standard instancing requires a fixed stride for per-instance attributes, which can lead to significant memory waste if instance data is sparse or variable-sized [11005]. By moving instance data into SSBOs, developers can utilize the GPU's random access capabilities to fetch only the necessary data for each instance, enabling techniques like bit-packing, dynamic allocation, and procedural generation of instance properties [S1, S3]. This approach is particularly prevalent in voxel rendering and particle systems where the number and properties of instances can change frequently [S1, S2, S3]. The technique relies on the availability of Shader Storage Buffer Objects (introduced in OpenGL 4.3) or equivalent structures in other APIs (e.g., `StructuredBuffer` in DirectX) [S1, S2, S3, 11005].

## Sources
[S1] Image-space sheet-buffered splatting on the GPU, https://openalex.org/W2419486389, fetched 2026-10-27 (unofficial)
