# GPU Instancing with Texture Buffer Objects

Also called: TBO Instancing

## Overview
GPU Instancing with Texture Buffer Objects is a technique for rendering large numbers of identical geometry primitives, such as voxels, by storing instance-specific attributes (position, scale) in a Texture Buffer Object (TBO) and reading them in a vertex shader [S1, unofficial]. It solves the performance bottleneck of transmitting per-instance data by sending base mesh data only once via a Vertex Buffer Object (VBO), while the TBO holds the variable instance parameters [S1, unofficial]. This method sets itself apart from standard instancing by leveraging the texture memory hierarchy for instance data, which can offer different caching characteristics or compatibility with specific shader models compared to standard uniform buffers or standard instancing arrays [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A base mesh definition (vertices, normals, colors) and a list of instance attributes (e.g., voxel positions).
*   **Output:** Rasterized voxels on the screen.

**Data Structures:**
1.  **Base VBO:** Contains the geometry for a single voxel (typically a cube).
    *   **Layout:**
        *   Vertices: 8 vertices, each 3 floats ($x, y, z$).
        *   Normals: 8 normals, each 3 floats ($x, y, z$).
        *   Colors/Textures: Additional floats as required by the shader.
    *   **Size Calculation:** $3 \times \text{sizeof(float)} \times 8$ (vertices) $+ 3 \times \text{sizeof(float)} \times 8$ (normals) $+ \text{floats for color/texture}$ [S1, unofficial].
2.  **Texture Buffer Object (TBO):** Stores instance data for all voxels.
    *   **Layout:** A contiguous array of instance attributes.
    *   **Content:** Typically 3 floats per voxel for position ($x, y, z$). Scale and other parameters are stored if necessary [S1, unofficial].
    *   **Size:** $3 \times \text{sizeof(float)} \times \text{number of voxels}$ [S1, unofficial].

**Steps:**
1.  **Create Base VBO:** Upload the base voxel geometry (vertices, normals, colors) to the GPU once. This data is static for all instances [S1, unofficial].
2.  **Create TBO:** Generate a Texture Buffer Object and populate it with the instance attributes (positions, scales) for all active voxels [S1, unofficial].
3.  **Vertex Shader Execution:**
    *   The vertex shader receives the base vertex data from the VBO.
    *   It reads the instance-specific data (position, scale) from the TBO using the instance ID.
    *   It applies the transformation (translation/scaling) to the base vertex to determine the final position of the instance [S1, unofficial].
4.  **Rendering:** Issue a draw call that renders the base mesh for the number of instances specified by the TBO content. Only the instance count or TBO reference is sent, not individual vertex data for each cube [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Setup Base VBO (Cube Geometry)
# 8 vertices, 8 normals, plus optional colors/textures
base_vbo_data = [
    # Vertices (8 * 3 floats)
    -0.5, -0.5,  0.5,  0.5, -0.5,  0.5,  0.5,  0.5,  0.5, -0.5,  0.5,  0.5,
    -0.5, -0.5, -0.5,  0.5, -0.5, -0.5,  0.5,  0.5, -0.5, -0.5,  0.5, -0.5,
    # Normals (8 * 3 floats)
    # ... (normals for each vertex) ...
    # Colors/Textures (if any)
]
base_vbo = create_vbo(base_vbo_data)

# 2. Setup TBO with Instance Data
# Example: 1000 voxels, each with 3 floats for position
num_voxels = 1000
instance_data = [
    # Position x, y, z for each voxel
    x1, y1, z1,
    x2, y2, z2,
    # ...
    xN, yN, zN
]
tbo = create_texture_buffer_object(instance_data)

# 3. Vertex Shader Logic (Conceptual)
# In the vertex shader:
# vec3 position = texture(tbo, gl_InstanceID).xyz;
# gl_Position = projection * view * (model * vec4(position, 1.0));

# 4. Render
bind_vbo(base_vbo)
bind_tbo(tbo)
draw_instanced(base_mesh, num_voxels)
```

**Parameters:**
*   **Base VBO Size:** Determined by the complexity of the single voxel mesh. For a simple cube with normals: $3 \times 4 \times 8 + 3 \times 4 \times 8 = 192$ bytes (excluding colors/textures) [S1, unofficial].
*   **Instance Data Size:** $12$ bytes per voxel (3 floats for position) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in the TBO may affect position accuracy for very large scenes.
*   **Boundary Conditions:** Ensure the TBO size matches the number of instances exactly to avoid out-of-bounds reads in the shader.
*   **Degenerate Input:** If the number of voxels is zero, the draw call should be skipped or handled gracefully to avoid rendering artifacts.

## Subtopics

### CPU-GPU Parallelization with Double Buffering
A parallelization strategy using two threads (CPU and GPU) and two TBOs to overlap geometry generation with rendering, switching buffers between frames [S1, unofficial]. The CPU thread checks the octree PVS and updates a TBO for instancing for the next frame, while the GPU thread renders the 2 passes (Depth-only and Color) using the TBO created by the CPU thread in the previous step [S1, unofficial]. After rendering, the system switches TBOs. If the camera has not moved, CPU calculations can be skipped entirely, reusing the previous TBO [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of instances, but the overhead per instance is minimal since the base geometry is sent once. The CPU overhead is reduced by parallelization strategies [S1, unofficial].
*   **Memory Complexity:** Memory usage is dominated by the TBO size, which scales linearly with the number of voxels ($12$ bytes per voxel for position alone) [S1, unofficial].
*   **Performance:** Significantly faster than sending individual vertex data for each voxel. Avoids expensive shader computations that are later overwritten by closer fragments (similar to Carmack's reverse in Quake3) [S1, unofficial].
*   **Strengths:** Efficient for rendering large numbers of identical objects. Allows for CPU-GPU parallelization.
*   **Limitations:** Requires careful management of TBO updates. Not suitable for highly dynamic per-vertex data that changes every frame without careful buffering.
*   **When to Use:** Ideal for voxel engines or any scenario with many identical primitives where instance data (position, scale) is the primary variable.
*   **When Not to Use:** When the geometry of each instance varies significantly, or when the number of instances is small enough that the overhead of TBO management outweighs the benefits.

## Background
This technique was discussed in the context of optimizing OpenGL voxel engines, where performance bottlenecks were identified in rendering large numbers of voxels [S1, unofficial]. The approach leverages existing OpenGL features (TBOs and Instancing) to achieve efficient rendering. It was proposed as a solution to the slowness of naive voxel rendering methods [S1, unofficial].

## Sources
[S1] OpenGL voxel engine slow, https://stackoverflow.com/questions/4563693/opengl-voxel-engine-slow, fetched 2026-10-26 (unofficial)
