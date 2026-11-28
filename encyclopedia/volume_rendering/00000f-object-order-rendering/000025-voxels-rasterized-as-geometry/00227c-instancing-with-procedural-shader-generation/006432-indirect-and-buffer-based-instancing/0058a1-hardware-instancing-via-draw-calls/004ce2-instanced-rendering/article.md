# Instanced Rendering

Also called: DrawIndexedInstanced, Geometry Instancing, Hardware Instancing, Hardware Instancing in DirectX 11, Instanced Rendering for Voxels

## Overview
Instanced rendering draws multiple copies of a base geometry with different per-instance attributes using a single draw call [S2, unofficial]. It solves the high CPU overhead and state change costs associated with rendering thousands of static objects individually [S4, unofficial][S6, unofficial]. It sets itself apart from standard object-order rendering by batching geometry and attribute updates, and from indirect instancing by relying on explicit draw calls rather than compute-driven dispatch [S2, unofficial][S3, unofficial].

## How it works
**Inputs:**
- Base geometry: A single mesh (e.g., a voxel cube) defined by vertices and indices [S3, unofficial]. For flat-shaded voxels, this typically involves duplicated vertices (e.g., 24 vertices and 36 indices for a cube) [S5, unofficial].
- Instance Data Buffer: A GPU-accessible buffer storing per-instance parameters such as position, scale, color, or transformation matrices [S3, unofficial][S4, unofficial][S5, unofficial].
- Instance Count: The number of instances to render [S5, unofficial].

**Outputs:**
- Rendered scene with multiple instances of the base geometry transformed according to the instance data [S3, unofficial][S6, unofficial].

**Data Structures:**
- **Inner Buffer:** Contains the base geometry vertices [S3, unofficial].
- **Outer Buffer:** Contains one vertex per instance with per-instance attributes [S3, unofficial].
- **Vertex Attribute Divisor:** A GPU state that determines how often the vertex shader advances to the next element of an instance attribute [S5, unofficial]. For per-instance data, the divisor is set to 1 [S3, unofficial][S5, unofficial].

**Steps:**
1. Update the vertex shader signature to include instance fields, assigning them to slot 1 or higher [S1, unofficial].
2. Bind the Vertex Array Object (VAO) for the base geometry [S5, unofficial].
3. Provide an inner vertex buffer containing the base geometry vertices (e.g., 8 vertices for a cube) [S3, unofficial].
4. Upload transformation matrices or per-instance attributes to an outer buffer [S4, unofficial][S6, unofficial].
5. Bind the instance buffer to the shader slot matching the signature [S1, unofficial].
6. Set the vertex attribute divisor to 1 for instance attributes so they advance once per instance [S3, unofficial][S5, unofficial].
7. Call an instanced draw function (e.g., `glDrawElementsInstanced`, `DrawIndexedInstanced`) specifying primitive type, count, type, offset, and instance count [S2, unofficial][S5, unofficial].

**Formula:**
The internal behavior of instancing is equivalent to looping through instance counts and incrementing `gl_InstanceID`:
$$
\text{for } (int\ n = 0;\ n < \text{instancecount};\ n++) \{ \\
\quad \text{glVertexAttrib1i(gl\_InstanceID, n);} \\
\quad \text{glDrawElements(mode, count, type, indices);} \\
\}
$$
[S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_instanced_voxels(base_mesh, instance_data, instance_count):
    # Inputs: base_mesh (dict with 'vertices' list, 'indices' list), 
    #         instance_data (list of per-instance structs), instance_count (int)
    
    # 1. Setup Base Geometry (Inner Buffer)
    # Bind base vertices to attribute location 0. Divisor 0 means per-vertex.
    gl_bind_buffer(GL_ARRAY_BUFFER, inner_vbo)
    gl_vertex_attrib_pointer(location=0, size=3, type=GL_FLOAT, normalized=False, stride=0, pointer=None)
    gl_vertex_attrib_divisor(location=0, divisor=0)
    gl_enable_vertex_attrib_array(location=0)
    
    # 2. Setup Instance Attributes (Outer Buffer)
    # Upload per-instance transformation matrices. 
    # OpenGL splits 4x4 matrices into 4 vec4 columns for attribute arrays.
    gl_bind_buffer(GL_ARRAY_BUFFER, outer_vbo)
    
    # Bind each matrix column to a separate attribute location.
    # Divisor 1 means the attribute advances once per instance, not per vertex.
    # stride=64 bytes (4 columns * 16 bytes/vec4)
    gl_vertex_attrib_pointer(location=1, size=4, type=GL_FLOAT, normalized=False, stride=64, pointer=0)
    gl_vertex_attrib_divisor(location=1, divisor=1)
    gl_enable_vertex_attrib_array(location=1)
    
    gl_vertex_attrib_pointer(location=2, size=4, type=GL_FLOAT, normalized=False, stride=64, pointer=16)
    gl_vertex_attrib_divisor(location=2, divisor=1)
    gl_enable_vertex_attrib_array(location=2)
    
    gl_vertex_attrib_pointer(location=3, size=4, type=GL_FLOAT, normalized=False, stride=64, pointer=32)
    gl_vertex_attrib_divisor(location=3, divisor=1)
    gl_enable_vertex_attrib_array(location=3)
    
    gl_vertex_attrib_pointer(location=4, size=4, type=GL_FLOAT, normalized=False, stride=64, pointer=48)
    gl_vertex_attrib_divisor(location=4, divisor=1)
    gl_enable_vertex_attrib_array(location=4)
    
    # 3. Execute Instanced Draw Call
    # The GPU iterates instance_count times. 
    # Internally equivalent to: for n in range(instance_count): gl_InstanceID = n; draw_elements(...)
    gl_draw_elements_instanced(
        mode=GL_TRIANGLES,
        count=len(base_mesh['indices']),
        type=GL_UNSIGNED_INT,
        first=0,
        instance_count=instance_count
    )
```

**Parameters:**
- **Divisor:** Set to 1 for per-instance attributes [S5, unofficial].
- **Instance Count:** Equal to the number of valid instances in the buffer (e.g., filtered by frustum) [S5, unofficial].

**Edge Cases and Pitfalls:**
- **Matrix Alignment:** When feeding a matrix into the shader, ensure correct column or row major alignment to avoid transformation errors [S1, unofficial].
- **Deprecated Features:** Using deprecated features like `gl_Vertex` and `gl_ModelViewProjectionMatrix` in modern OpenGL is bad practice [S2, unofficial].
- **Uniform Limits:** Using uniforms to store per-instance data via `gl_InstanceID` is slower than using vertex attributes/VBOs and is limited by `MAX_VERTEX_UNIFORM_COMPONENTS` (typically 512-1024 floats), which is insufficient for tens of thousands of instances [S2, unofficial].
- **Driver Dependency:** Performance benefit depends on the graphics driver [S4, unofficial].

## Subtopics

### GPU Instancing with MaterialPropertyBlock
A Unity variant that allows different materials per instance [S7, unofficial]. Method 1 (Lowest Efficiency) generates multiple prefabs with separate MaterialPropertyBlocks [S7, unofficial]. Method 2 (Medium Efficiency) calls `DrawMesh` multiple times with different MaterialPropertyBlocks [S7, unofficial]. Method 3 (Highest Efficiency) uses `DrawMeshInstanced` with an array of MaterialPropertyBlocks to set different materials per mesh instance [S7, unofficial]. Multi-pass rendering allows control over draw order but does not support instancing [S7, unofficial].

### Animation via Virtual Instancing
An animation technique that avoids rebuilding acceleration structures (AS) by updating transformation matrices or instance rotations in shaders [S8]. For proxy geometry transformation, transformation matrices inside µLAS are updated; rebuilding µLAS is fast and non-blocking [S8]. Mesostructures and nanostructures adjust automatically without AS updates because instances are instantiated virtually [S8]. For protein instance jittering, instance rotation is manipulated based on time to compute transformation matrices $M_2$, inducing animation at the nLAS level [S8]. This computation is done in rendering shaders, requiring no AS update or rebuild [S8].

## Operating characteristics
- **Time Complexity:** Reduces CPU overhead from $O(N)$ draw calls to $O(1)$ draw call for $N$ instances [S2, unofficial][S6, unofficial].
- **Memory Complexity:** Requires additional memory for the instance data buffer, proportional to the number of instances [S3, unofficial].
- **Performance:** Significant performance gain for large voxel scenes with many static objects [S4, unofficial][S6, unofficial]. Performance depends on GPU driver and scene complexity [S4, unofficial].
- **Strengths:** Avoids repeated matrix multiplications, inverses, uniform uploads, and texture state changes per frame [S6, unofficial].
- **Limitations:** Using a matrix for cube alignment incurs a GPU draw penalty rather than a CPU penalty [S1, unofficial]. Using uniforms for per-instance data is limited by `MAX_VERTEX_UNIFORM_COMPONENTS` [S2, unofficial].
- **Scaling:** Not given in the sources.

## Background
Instanced rendering originated as a technique to optimize rendering of large numbers of static objects like terrain chunks, leaves, and trees [S5, unofficial]. It was developed to address performance bottlenecks of traditional per-object rendering where each object requires a separate draw call and state change [S2, unofficial][S6, unofficial]. The technique leverages GPU hardware capabilities to iterate over instance data efficiently [S1, unofficial][S2, unofficial]. It is widely used in voxel engines and large-scale scene rendering [S3, unofficial][S4, unofficial]. Specific origin details (author, year, or primary paper) are not given in the sources.

## Sources
[S1] Implement hardware instancing on Directx11, https://stackoverflow.com/questions/55765822/implement-hardware-instancing-on-directx11, fetched 2026-10-26 (unofficial)
[S2] How to do instancing the right way in OpenGL., https://stackoverflow.com/questions/21539234/how-to-do-instancing-the-right-way-in-opengl, fetched 2026-10-26 (unofficial)
[S3] How to create more vertexes from within a shader in OpenGL?, https://gamedev.stackexchange.com/questions/72442/how-to-create-more-vertexes-from-within-a-shader-in-opengl, fetched 2026-10-26 (unofficial)
[S4] OpenGL How to render a voxel based world scene with many objects, https://stackoverflow.com/questions/31256095/opengl-how-to-render-a-voxel-based-world-scene-with-many-objects, fetched 2026-10-27 (unofficial)
[S5] MProgramingUnityCsharp/VoxelCoreEngine, https://github.com/MProgramingUnityCsharp/VoxelCoreEngine, fetched 2026-10-27 (unofficial)
[S6] OpenGL How to render a voxel based world scene with many objects, https://www.stackoverflow.com/questions/31256095, fetched 2026-10-28 (unofficial)
[S7] ShuchangXu/VolumeCloud, https://github.com/ShuchangXu/VolumeCloud, fetched 2026-10-26 (unofficial)
[S8] Nanouniverse: Virtual Instancing of Structural Detail and Adaptive Shell Mapping., https://doi.org/10.1109/tvcg.2025.3618914, fetched 2026-10-26
