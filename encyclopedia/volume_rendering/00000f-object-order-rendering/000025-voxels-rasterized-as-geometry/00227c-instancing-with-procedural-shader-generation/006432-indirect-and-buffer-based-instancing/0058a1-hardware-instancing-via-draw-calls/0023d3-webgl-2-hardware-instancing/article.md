# WebGL 2 Hardware Instancing

Also called: Cube Particles, Hardware Instancing, Instanced cube particles, gl.drawElementsInstanced

## Overview
WebGL 2 Hardware Instancing is a rendering technique that sends a single primitive (typically a 2x2x2 cube) to the GPU and executes `gl.drawElementsInstanced` to render hundreds of thousands of copies simultaneously [S2, unofficial]. It bypasses CPU bottlenecks by packing translation, scaling, and color data into flat `Float32Arrays` and uploading them to the GPU, allowing the vertex shader to apply per-instance transformations [S2, unofficial]. It sets itself apart from standard instancing by often utilizing specific attribute layouts for voxel-like data (e.g., separate scale and position attributes) rather than full transformation matrices, and differs from indirect draw calls by relying on a fixed instance count provided directly by the CPU [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Base geometry (vertices, normals, indices), instance data (positions, scales, colors), and shader programs [S1, unofficial][S2, unofficial].
*   **Output:** Rasterized voxels on the screen [S2, unofficial].

**Data Structures:**
1.  **Base Geometry:** A single cube defined by 24 vertices (4 per face for flat shading) and 36 indices [S1, unofficial].
    *   `positions`: Array of 24 `vec3` coordinates [S2, unofficial].
    *   `normals`: Array of 24 `vec3` normals [S2, unofficial].
    *   `indices`: Array of 36 `uint16` or `uint32` indices [S2, unofficial].
2.  **Instance Data Buffer:** A GPU buffer storing per-instance attributes. The layout varies by implementation:
    *   **Variant A (Voxel Terrain):** Separate attributes for position (`vec3`), scale (`float`), and color (`vec3`) [S2, unofficial].
    *   **Variant B (Particles):** A packed structure of 21 floats per instance: 16 floats for a transformation matrix, 4 floats for color, and 1 float for health [S3, unofficial].

**Numbered Steps:**
1.  **Create Base Geometry:** Generate buffers for vertex positions, normals, and indices. Upload them using `gl.bufferData` with `gl.STATIC_DRAW` [S1, unofficial][S2, unofficial].
2.  **Prepare Instance Data:** On the CPU rendering thread, update instance parameters (position, scale, color) for each frame [S1, unofficial]. Create flat `Float32Arrays` for these attributes. For a grid of size $N \times N$, the number of instances is $N^2$ [S2, unofficial].
3.  **Upload Instance Data:** For each instance attribute (e.g., position, scale, color):
    *   Create a buffer object (`gl.createBuffer`) [S2, unofficial].
    *   Bind it to `gl.ARRAY_BUFFER` [S2, unofficial].
    *   Upload data using `gl.bufferData` with `gl.DYNAMIC_DRAW` or `gl.STATIC_DRAW` depending on update frequency [S1, unofficial].
    *   Enable the vertex attribute array using `gl.enableVertexAttribArray` [S1, unofficial].
    *   Set the pointer using `gl.vertexAttribPointer` to define the stride and offset [S1, unofficial].
    *   Set the divisor to 1 using `gl.vertexAttribDivisor` to indicate that this attribute changes once per instance, not once per vertex [S1, unofficial].
4.  **Vertex Shader Processing:** The vertex shader receives the base vertex and instance attributes. It applies the instance data to the base vertex position. For Variant A, the transformation is typically:
    $$ pos.y \ *= \ aInstanceScaleY $$
    $$ pos.xyz \ += \ aInstancePosition $$
    [S2, unofficial]
5.  **Draw Call:** Execute `gl.drawElementsInstanced` with the base index count and the number of instances [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Setup Base Geometry
positions = [-1,-1,1, 1,-1,1, 1,1,1, -1,1,1, -1,-1,-1, -1,1,-1, 1,1,-1, 1,-1,-1, -1,1,-1, -1,1,1, 1,1,1, 1,1,-1, 1,-1,-1, 1,-1,1, -1,-1,-1, -1,-1,1, -1,1,1, -1,1,-1, 1,-1,-1, 1,-1,1, 1,1,1, -1,-1,-1, -1,-1,1, -1,1,1, -1,1,-1]
normals = [0,0,1, 0,0,1, 0,0,1, 0,0,1, 0,0,-1, 0,0,-1, 0,0,-1, 0,0,-1, 0,1,0, 0,1,0, 0,1,0, 0,1,0, 0,-1,0, 0,-1,0, 0,-1,0, 0,-1,0, 1,0,0, 1,0,0, 1,0,0, 1,0,0, -1,0,0, -1,0,0, -1,0,0, -1,0,0]
indices = [0,1,2, 0,2,3, 4,5,6, 4,6,7, 8,9,10, 8,10,11, 12,13,14, 12,14,15, 16,17,18, 16,18,19, 20,21,22, 20,22,23]

pos_buffer = create_buffer()
bind_buffer(ARRAY_BUFFER, pos_buffer)
buffer_data(ARRAY_BUFFER, list(positions), STATIC_DRAW)

norm_buffer = create_buffer()
bind_buffer(ARRAY_BUFFER, norm_buffer)
buffer_data(ARRAY_BUFFER, list(normals), STATIC_DRAW)

index_buffer = create_buffer()
bind_buffer(ELEMENT_ARRAY_BUFFER, index_buffer)
buffer_data(ELEMENT_ARRAY_BUFFER, list(indices), STATIC_DRAW)

# 2. Setup Instance Attributes
grid_size = 500
spacing = 2
num_instances = grid_size * grid_size
instance_positions = [0.0] * (num_instances * 3)
instance_scales = [0.0] * num_instances
instance_colors = [0.0] * (num_instances * 3)

offset = (grid_size * spacing) / 2
for z in range(grid_size):
    for x in range(grid_size):
        idx = z * grid_size + x
        instance_positions[idx * 3] = (x * spacing) - offset
        instance_positions[idx * 3 + 1] = 0.0
        instance_positions[idx * 3 + 2] = (z * spacing) - offset
        pixel_index = idx * 4
        instance_scales[idx] = 1.0 + (height_data[pixel_index] / 255.0) * 100.0
        instance_colors[idx * 3] = color_data[pixel_index] / 255.0
        instance_colors[idx * 3 + 1] = color_data[pixel_index + 1] / 255.0
        instance_colors[idx * 3 + 2] = color_data[pixel_index + 2] / 255.0

pos_inst_buffer = create_buffer()
bind_buffer(ARRAY_BUFFER, pos_inst_buffer)
buffer_data(ARRAY_BUFFER, instance_positions, DYNAMIC_DRAW)
enable_vertex_attrib_array(aInstancePosition)
vertex_attrib_pointer(aInstancePosition, 3, FLOAT, False, 0, 0)
vertex_attrib_divisor(aInstancePosition, 1)

scale_inst_buffer = create_buffer()
bind_buffer(ARRAY_BUFFER, scale_inst_buffer)
buffer_data(ARRAY_BUFFER, instance_scales, DYNAMIC_DRAW)
enable_vertex_attrib_array(aInstanceScaleY)
vertex_attrib_pointer(aInstanceScaleY, 1, FLOAT, False, 0, 0)
vertex_attrib_divisor(aInstanceScaleY, 1)

color_inst_buffer = create_buffer()
bind_buffer(ARRAY_BUFFER, color_inst_buffer)
buffer_data(ARRAY_BUFFER, instance_colors, DYNAMIC_DRAW)
enable_vertex_attrib_array(aInstanceColor)
vertex_attrib_pointer(aInstanceColor, 3, FLOAT, False, 0, 0)
vertex_attrib_divisor(aInstanceColor, 1)

# 3. Draw
draw_elements_instanced(TRIANGLES, 36, UNSIGNED_SHORT, 0, num_instances)
```

**Parameters:**
*   `gridSize`: The dimension of the square world grid. A value of 500 results in 250,000 cubes ($500 \times 500$) [S2, unofficial].
*   `spacing`: The distance between voxel centers. A typical value is 2 [S2, unofficial].
*   `max_instances`: The maximum number of instances supported by the buffer allocation is 65536, calculated from the buffer size `sizeof(glm::vec2)*36*65536` for texture coordinates and `sizeof(glm::vec3)*65536` for positions [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Using `Float32Array` for large coordinates may lead to z-fighting or jitter if the world coordinates exceed the precision limits of 32-bit floats [G]. Use double-precision shaders or offset the world origin if necessary.
*   **Boundary Conditions:** When sampling heightmaps or color maps, ensure pixel indices are clamped to the image dimensions to avoid out-of-bounds access [S2, unofficial].
*   **Attribute Divisor:** If the divisor for a texture coordinate attribute is not set to 1 (default is 0), the shader will only read the first 36 texture coordinates for all instances, causing the same texture to appear on all blocks [S1, unofficial].

## Subtopics

### Instanced Drawing and Batch Merging
Merges terrain tiles with the same Level of Detail (LOD) level into a single draw call using instanced drawing to reduce CPU overhead from issuing many draw calls [S4]. It passes the tile’s world transformation matrix and texture atlas offset as instance data for each instance [S4]. The number of draw calls is reduced from $M$ (total visible tiles) to the number of distinct LOD levels (typically no more than 10), where $M$ has LOD level distribution $c_0, c_1, \dots, c_{max}$ [S4].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of instances, as the CPU must prepare the instance data buffer each frame [S1, unofficial]. The GPU processing time is also proportional to $N$ [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for the instance data buffer. For Variant A, this is $8 \times N$ bytes (3 floats for pos, 1 for scale, 3 for color) [S2, unofficial]. For Variant B, it is $84 \times N$ bytes (21 floats) [S3, unofficial].
*   **Measured Performance:** Rendering 250,000 cubes (500x500 grid) is feasible in real-time on modern GPUs [S2, unofficial].
*   **Strengths:** Bypasses CPU bottlenecks associated with issuing individual draw calls for each voxel [S2, unofficial]. Allows for massive voxel counts [S2, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
This technique is widely used in WebGL-based voxel games and terrain rendering [S2, unofficial]. It was popularized by projects like "CubeWorld-Canyon" which demonstrated rendering large terrains using `gl.drawElementsInstanced` [S2, unofficial]. It is also used in particle systems, such as "CubeEngine," where cubes are rendered as particles with instancing [S3, unofficial]. The concept of batch merging for terrain tiles using instancing is discussed in WebGPU literature, indicating its applicability to next-generation web graphics APIs [S4].

## Sources
[S1] Only first vertex of data used in shader OpenGL, https://gamedev.stackexchange.com/questions/135416/only-first-vertex-of-data-used-in-shader-opengl, fetched 2026-10-26 (unofficial)
[S2] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S3] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
[S4] Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU, https://doi.org/10.1145/3844212.3844228, fetched 2026-11-02
