# Hardware Instancing via Draw Calls

Also called: GPU Instancing, Instanced Rendering

## Overview
Hardware Instancing via Draw Calls is a rendering technique that renders a large number of identical meshes (voxels) efficiently by issuing a single draw call that instructs the GPU to replicate a base primitive multiple times [S1, unofficial]. It solves the CPU bottleneck associated with issuing individual draw calls for large numbers of voxels by decoupling the CPU-side draw call issuance from the GPU-side execution logic [S1, unofficial]. Unlike standard per-object rendering, which requires repeated matrix multiplications, uniform uploads, and state changes per frame, this method uploads transformation data once and lets the GPU handle the replication [S1, unofficial].

## Shared principles
The family relies on the GPU's ability to iterate over a base geometry and apply per-instance attributes (such as position, scale, or color) defined in a buffer. The core mechanism involves:
1.  **Base Geometry:** A single mesh (e.g., a voxel cube) is defined with vertices and indices. For voxels, this often includes duplicated vertices for flat shading (e.g., 24 vertices and 36 indices for a cube) .
2.  **Instance Data Buffer:** A GPU-accessible buffer (e.g., `StructuredBuffer`, `SSBO`, or VBO) stores per-instance parameters. Common data includes position (x, y, z), occupancy flags, scale, and color [S1, unofficial].
3.  **Instanced Draw Call:** A specific API call (e.g., `gl.drawElementsInstanced`, `Graphics.DrawMeshInstancedIndirect`) is used. This call takes the base geometry, the instance buffer, and an instance count. The GPU automatically iterates through the instance buffer, applying the per-instance data to the base vertices during vertex shader execution [S1, unofficial].
4.  **Vertex Shader Processing:** The vertex shader accesses the instance data using a built-in instance ID (e.g., `gl_InstanceID` or `unity_InstanceID`) to transform the base vertex position into world space .

## Subtopics

### [GPU Instancing with Indirect Draw Calls](2205-gpu-instancing-with-indirect-draw-calls.md)
GPU Instancing with Indirect Draw Calls is a technique to render a large number of identical meshes (voxels) efficiently by using indirect draw calls. Instead of issuing one draw call per instance, a single draw call is issued that reads instance counts and parameters from a GPU-accessible buffer, allowing the GPU to generate the draw commands itself. It is distinguished from its siblings by decoupling the CPU-side draw call issuance from the GPU-side execution logic, enabling dynamic instance counts and reducing CPU-GPU synchronization overhead compared to standard hardware instancing. See [GPU Instancing with Indirect Draw Calls](2205-gpu-instancing-with-indirect-draw-calls.md).

### [WebGL 2 Hardware Instancing](3152-webgl-2-hardware-instancing.md)
WebGL 2 Hardware Instancing is a rendering technique that sends a single base primitive (typically a 2x2x2 cube) to the GPU and executes `gl.drawElementsInstanced` to render hundreds of thousands of copies simultaneously. It solves the CPU bottleneck associated with issuing individual draw calls for large numbers of voxels by packing translation, scaling, and color data into flat typed arrays. Unlike surface-based instancing which modulates geometry with noise, this method relies on explicit instance buffers and procedural sampling to dictate vertex positions and colors. See [WebGL 2 Hardware Instancing](3152-webgl-2-hardware-instancing.md).

### Instanced Rendering
**Definition:** A rendering technique where a single vertex object is uploaded once, and transformation matrices are uploaded in a buffer, allowing efficient drawing of many static objects with the same geometry [S1, unofficial].
**Steps:**
1.  Assume all voxels use the same geometry and are mostly static [S1, unofficial].
2.  Upload transformation matrices in a buffer [S1, unofficial].
3.  Upload the vertex object once [S1, unofficial].
**Comparison:** Compared to per-object rendering, instanced rendering avoids repeated matrix multiplications, inverses, uniform uploads, and texture state changes per frame, which are expensive operations [S1, unofficial].

## Background
The technique originated as a solution to the performance limitations of traditional immediate mode rendering, where each object required a separate draw call. Early implementations focused on static geometry with shared materials. The method has since evolved to support dynamic instance data via GPU buffers and indirect command buffers, allowing for complex, large-scale voxel scenes and particle systems. It is widely used in voxel-based games, terrain rendering, and particle effects where thousands of identical or similar primitives are rendered [S1, unofficial].

## Sources
[S1] OpenGL How to render a voxel based world scene with many objects, https://www.stackoverflow.com/questions/31256095, fetched 2026-10-28 (unofficial)
