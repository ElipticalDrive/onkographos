# WebGL 2 Hardware Instancing

Also called: Cube Particles, Hardware Instancing, Instanced cube particles, gl.drawElementsInstanced

## Overview
WebGL 2 Hardware Instancing is a rendering technique that sends a single primitive (typically a 2x2x2 cube) to the GPU and executes `gl.drawElementsInstanced` to render hundreds of thousands of copies simultaneously [S1, unofficial]. It bypasses CPU bottlenecks by packing translation, scaling, and color data into flat `Float32Arrays` and uploading them to the GPU, allowing the vertex shader to apply per-instance transformations [S1, unofficial]. It sets itself apart from standard instancing by often utilizing specific attribute layouts for voxel-like data (e.g., separate scale and position attributes) rather than full transformation matrices, and differs from indirect draw calls by relying on a fixed instance count provided directly by the CPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Base geometry (vertices, normals, indices), instance data (positions, scales, colors), and shader programs.
*   **Output:** Rasterized voxels on the screen.

**Data Structures:**
1.  **Base Geometry:** A single cube defined by 24 vertices (4 per face for flat shading) and 36 indices [S1, unofficial].
    *   `positions`: Array of 24 `vec3` coordinates.
    *   `normals`: Array of 24 `vec3` normals.
    *   `indices`: Array of 36 `uint16` or `uint32` indices.
2.  **Instance Data Buffer:** A GPU buffer storing per-instance attributes. The layout varies by implementation:
    *   **Variant A (Voxel Terrain):** Separate attributes for position (`vec3`), scale (`float`), and color (`vec3`) [S1, unofficial].
    *   **Variant B (Particles):** A packed structure of 21 floats per instance: 16 floats for a transformation matrix, 4 floats for color, and 1 float for health [S2, unofficial].

**Numbered Steps:**
1.  **Create Base Geometry:** Generate buffers for vertex positions, normals, and indices. Upload them using `gl.bufferData` with `gl.STATIC_DRAW` [S1, unofficial].
2.  **Prepare Instance Data:** On the CPU rendering thread, update instance parameters (position, scale, color) for each frame [S1, unofficial]. Create flat `Float32Array`s for these attributes. For a grid of size $N \times N$, the number of instances is $N^2$ [S1, unofficial].
3.  **Upload Instance Data:** For each instance attribute (e.g., position, scale, color):
    *   Create a buffer object (`gl.createBuffer`).
    *   Bind it to `gl.ARRAY_BUFFER`.
    *   Upload data using `gl.bufferData` with `gl.DYNAMIC_DRAW` or `gl.STATIC_DRAW` depending on update frequency [S1, unofficial].
    *   Enable the vertex attribute array using `gl.enableVertexAttribArray`.
    *   Set the pointer using `gl.vertexAttribPointer` to define the stride and offset.
    *   Set the divisor to 1 using `gl.vertexAttribDivisor` to indicate that this attribute changes once per instance, not once per vertex [S1, unofficial].
4.  **Vertex Shader Processing:** The vertex shader receives the base vertex and instance attributes. It applies the instance data to the base vertex position. For Variant A, the transformation is typically:
    $$ pos.y \ *= \ aInstanceScaleY $$
    $$ pos.xyz \ += \ aInstancePosition $$
    [S1, unofficial]
5.  **Draw Call:** Execute `gl.drawElementsInstanced` with the base index count and the number of instances [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Setup Base Geometry
positions = [-1,-1,1, 1,-1,1, 1,1,1, -1,1,1, ...] # 24 vertices
normals = [0,0,1, 0,0,1, ...] # 24 normals
indices = [0,1,2, 0,2,3, ...] # 36 indices

posBuffer = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, posBuffer)
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(positions), gl.STATIC_DRAW)

normBuffer = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, normBuffer)
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(normals), gl.STATIC_DRAW)

indexBuffer = gl.createBuffer()
gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, indexBuffer)
gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, new Uint16Array(indices), gl.STATIC_DRAW)

# 2. Setup Instance Attributes
numInstances = gridSize * gridSize # e.g., 500 * 500 = 250,000
instancePositions = new Float32Array(numInstances * 3)
instanceScales = new Float32Array(numInstances)
instanceColors = new Float32Array(numInstances * 3)

# Populate instance data (e.g., from heightmap)
offset = (gridSize * spacing) / 2
for x in range(gridSize):
    for z in range(gridSize):
        idx = (z * gridSize + x)
        instancePositions[idx * 3] = (x * spacing) - offset
        instancePositions[idx * 3 + 1] = 0 # Y is handled by scale/height
        instancePositions[idx * 3 + 2] = (z * spacing) - offset
        instanceScales[idx] = 1 + (heightData[pixelIndex] / 255.0) * 100
        instanceColors[idx * 3] = colorData[pixelIndex] / 255.0
        # ...

# Upload Position
posInstBuffer = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, posInstBuffer)
gl.bufferData(gl.ARRAY_BUFFER, instancePositions, gl.DYNAMIC_DRAW)
gl.enableVertexAttribArray(aInstancePosition)
gl.vertexAttribPointer(aInstancePosition, 3, gl.FLOAT, False, 0, 0)
gl.vertexAttribDivisor(aInstancePosition, 1)

# Upload Scale
scaleInstBuffer = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, scaleInstBuffer)
gl.bufferData(gl.ARRAY_BUFFER, instanceScales, gl.DYNAMIC_DRAW)
gl.enableVertexAttribArray(aInstanceScaleY)
gl.vertexAttribPointer(aInstanceScaleY, 1, gl.FLOAT, False, 0, 0)
gl.vertexAttribDivisor(aInstanceScaleY, 1)

# Upload Color
colorInstBuffer = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, colorInstBuffer)
gl.bufferData(gl.ARRAY_BUFFER, instanceColors, gl.DYNAMIC_DRAW)
gl.enableVertexAttribArray(aInstanceColor)
gl.vertexAttribPointer(aInstanceColor, 3, gl.FLOAT, False, 0, 0)
gl.vertexAttribDivisor(aInstanceColor, 1)

# 3. Draw
gl.drawElementsInstanced(gl.TRIANGLES, 36, gl.UNSIGNED_SHORT, 0, numInstances)
```

**Parameters:**
*   `gridSize`: The dimension of the square world grid. A value of 500 results in 250,000 cubes ($500 \times 500$) [S1, unofficial].
*   `spacing`: The distance between voxel centers. A typical value is 2 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Using `Float32Array` for large coordinates may lead to z-fighting or jitter if the world coordinates exceed the precision limits of 32-bit floats. [G] Use double-precision shaders or offset the world origin if necessary.
*   **Boundary Conditions:** When sampling heightmaps or color maps, ensure pixel indices are clamped to the image dimensions to avoid out-of-bounds access [S1, unofficial].
*   **Attribute Divisor:** Forgetting to call `gl.vertexAttribDivisor` or setting it to 0 will cause the instance data to be treated as per-vertex data, leading to incorrect rendering [S1, unofficial].

## Subtopics

### Instanced Drawing and Batch Merging
Merges terrain tiles with the same Level of Detail (LOD) level into a single draw call using instanced drawing to reduce CPU overhead from issuing many draw calls [S3]. It passes the tile’s world transformation matrix and texture atlas offset as instance data for each instance [S3]. The number of draw calls is reduced from $M$ (total visible tiles) to the number of distinct LOD levels (typically no more than 10), where $M$ has LOD level distribution $c_0, c_1, \dots, c_{max}$ [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of instances, as the CPU must prepare the instance data buffer each frame [S1, unofficial]. The GPU processing time is also proportional to $N$.
*   **Memory Complexity:** $O(N)$ for the instance data buffer. For Variant A, this is $8 \times N$ bytes (3 floats for pos, 1 for scale, 3 for color). For Variant B, it is $84 \times N$ bytes (21 floats) [S1, S2, unofficial].
*   **Measured Performance:** Rendering 250,000 cubes (500x500 grid) is feasible in real-time on modern GPUs [S1, unofficial].
*   **Strengths:** Bypasses CPU bottlenecks associated with issuing individual draw calls for each voxel [S1, unofficial]. Allows for massive voxel counts.
*   **Limitations:** Requires CPU-side update of instance data every frame if dynamic [S1, unofficial]. Not suitable for highly dynamic individual transformations without significant CPU overhead.
*   **When to Use:** Static or semi-static voxel terrains, particle systems with known bounds.
*   **When Not to Use:** Highly dynamic scenes with frequent individual vertex changes that cannot be batched.

## Background
This technique is widely used in WebGL-based voxel games and terrain rendering. It was popularized by projects like "CubeWorld-Canyon" which demonstrated rendering large terrains using `gl.drawElementsInstanced` [S1, unofficial]. It is also used in particle systems, such as "CubeEngine," where cubes are rendered as particles with instancing [S2, unofficial]. The concept of batch merging for terrain tiles using instancing is discussed in WebGPU literature, indicating its applicability to next-generation web graphics APIs [S3].

## Sources
[S1] evjapont-hash/CubeWorld-Canyon, https://github.com/evjapont-hash/CubeWorld-Canyon, fetched 2026-10-26 (unofficial)
[S2] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
[S3] Efficient Parsing and Rendering of Large-Scale Terrain Data Based on WebGPU, https://doi.org/10.1145/3844212.3844228, fetched 2026-11-02
