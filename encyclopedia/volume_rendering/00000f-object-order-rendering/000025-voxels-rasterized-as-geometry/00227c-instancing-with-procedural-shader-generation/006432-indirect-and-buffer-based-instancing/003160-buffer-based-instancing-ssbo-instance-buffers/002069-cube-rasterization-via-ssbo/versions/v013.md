# Cube Rasterization via SSBO

Also called: GL_DRAW_ELEMENTS instancing, GPU Instancing, Voxel Instance Rendering, glDrawElementsInstanced Voxel Draw

## Overview
This method renders $N$ voxels as instances of a base unit cube geometry using a single `glDrawElementsInstanced` draw call, where each instance is transformed by an offset and colored via a palette texture lookup [S1, unofficial]. It rasterizes 3 faces of each cube per voxel using a large index buffer and a large SSBO containing voxel data [S2, unofficial]. It is distinguished from point-sprite or ray-traced voxel methods by using explicit triangle geometry for exact cube shapes and procedural face culling based on camera orientation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of $N$ voxels, each defined by integer grid coordinates $(x, y, z)$ and a material/color index.
*   **Output:** A rasterized image of the voxel scene.

**Data Structures:**
1.  **Base Geometry (VAO):**
    *   Contains vertex attributes for a unit cube.
    *   Layout per vertex: `aLocalPosition` (vec3, float) and `aNormal` (vec3, float) [S1, unofficial].
    *   Stride: `CubeGeometry.FLOATS_PER_VERTEX * 4` bytes [S1, unofficial].
2.  **Instance Buffer (SSBO):**
    *   Bound as a `std430` buffer at binding point 0 [S2, unofficial].
    *   Layout: One `uvec2` per voxel [S2, unofficial].
    *   `data.x` (uvec2 component): Packs position. X: 11 bits, Y: 10 bits, Z: 11 bits [S2, unofficial].
    *   `data.y` (uvec2 component): Packs material properties. ALBEDO (16 bits, R5G6B5), METAL (4 bits), ROUGH (4 bits), INTENSITY (4 bits), TYPE (4 bits) [S2, unofficial].
3.  **Index Buffer (EBO):**
    *   A single large static buffer containing indices for all instances [S2, unofficial].
    *   Total indices: $N \times 18$ (3 faces $\times$ 2 triangles/face $\times$ 3 vertices/triangle) [S2, unofficial].
    *   Indices are pre-computed: `indices[i] = cube_indices[cube_local] + cube * num_cube_vertices`, where `num_cube_vertices = 8` [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Create the base unit cube VAO with position and normal attributes [S1, unofficial].
    *   Create the SSBO and populate it with voxel data packed into `uvec2` arrays [S2, unofficial].
    *   Create the large EBO with 18 indices per instance [S2, unofficial].
    *   Bind the VAO, EBO, and SSBO [S1, unofficial].
2.  **Draw Call:**
    *   Issue `glDrawElementsInstanced(GL_TRIANGLES, index_count, GL_UNSIGNED_SHORT, 0, instance_count)` [S1, unofficial].
3.  **Vertex Shader Execution (Per Vertex):**
    *   Determine the instance index from `gl_VertexID`: `instance = gl_VertexID >> 3` [S2, unofficial].
    *   Fetch voxel data from the SSBO using `instance` [S2, unofficial].
    *   Unpack position and albedo from the `uvec2` data using bitmasks [S2, unofficial].
    *   Compute world position from the unpacked coordinates [S2, unofficial].
    *   Determine the local camera position: `local_camera_pos = camera_origin - world_position` [S2, unofficial].
    *   Determine the cube face orientation based on `local_camera_pos`:
        *   Initialize `xyz` from `gl_VertexID` bits: `x = gl_VertexID & 1`, `y = (gl_VertexID & 2) >> 1`, `z = (gl_VertexID & 4) >> 2` [S2, unofficial].
        *   If `local_camera_pos.x > 0`, flip `xyz.x` (`xyz.x = 1 - xyz.x`) [S2, unofficial].
        *   If `local_camera_pos.y > 0`, flip `xyz.y` (`xyz.y = 1 - xyz.y`) [S2, unofficial].
        *   If `local_camera_pos.z > 0`, flip `xyz.z` (`xyz.z = 1 - xyz.z`) [S2, unofficial].
    *   Transform the vertex:
        *   `uvw = vec3(xyz)` [S2, unofficial].
        *   `local_pos = uvw * 2.0 - 1.0` [S2, unofficial].
        *   `world_pos = local_pos * 0.05 + world_position` [S2, unofficial].
        *   `gl_Position = view_projection_matrix * vec4(world_pos, 1.0)` [S2, unofficial].
    *   Output color (albedo) [S2, unofficial].

**Parameters:**
*   `num_cube_indices`: $N \times 3 \times 2 \times 3$ [S2, unofficial].
*   `MIN_ZOOM`: 0.3f [S1, unofficial].
*   `MAX_ZOOM`: 3.0f [S1, unofficial].
*   `ROTATION_SENSITIVITY`: 0.4f [S1, unofficial].
*   `AMBIENT_INTENSITY`: 0.35f [S1, unofficial].
*   `SPECULAR_INTENSITY`: 0.5f [S1, unofficial].
*   `SHININESS`: 32f [S1, unofficial].
*   `LIGHT_POS_WORLD`: {10f, 15f, 10f} [S1, unofficial].
*   `cameraDistance`: 5f [S1, unofficial].
*   `FPS_UPDATE_INTERVAL_NANOS`: 500,000,000 ns [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **VAO Availability:** VAOs are not universally available via pure GLES 3.0 on Android; explicit VBO/EBO binding may be required every frame, though overhead is negligible due to the single draw call [S1, unofficial].
*   **Zoom Clamping:** The zoom factor must be clamped to `[MIN_ZOOM, MAX_ZOOM]` to prevent extreme scaling artifacts [S1, unofficial].
*   **Rotation Direction:** Rotation direction depends on drag direction across screen halves (right/left), handled naturally by the sign of `dx` [S1, unofficial].
*   **Precision:** Position packing uses limited bits (11/10/11), which may limit grid resolution or precision for very large scenes [S2, unofficial].

## Subtopics

### [SVO Node Visualization via Instanced Cubes](16506-svo-node-visualization-via-instanced-cubes.md)
A compute-to-draw pipeline that converts SVO nodes into instanced cubes for visualization. A compute shader (mVisualCS) populates a VisualPackage buffer and an atomic counter, which is then used to render instanced cubes via indexed draw calls.

## Operating characteristics
*   **Time Complexity:** $O(1)$ draw call overhead plus $O(N)$ vertex processing for $N$ instances [S1, unofficial].
*   **Memory Complexity:** Requires an SSBO of size $2 \times N \times 4$ bytes (for two `uvec2`s) and an EBO of size $18 \times N \times 2$ bytes (assuming `GL_UNSIGNED_SHORT`) [S2, unofficial].
*   **Performance:** On dedicated graphics cards, `MODE_CUBE` is slower than `MODE_QUAD` (point sprite/ray-traced) by approximately a factor of two [S2, unofficial]. On Intel Integrated graphics, performance is similar [S2, unofficial].
*   **Strengths:** Provides exact cube geometry and explicit face culling logic, avoiding the artifacts associated with point sprites or ray-traced approximations [S2, unofficial].
*   **Limitations:** Higher memory bandwidth usage due to the large index buffer and instance buffer compared to point-sprite methods [S2, unofficial].

## Background
This method was implemented in an Android OpenGL ES 3.0 voxel visualization project [S1, unofficial]. It was also used in a high-performance voxel renderer experiment by Clay John, where it was compared against a point-sprite/ray-traced mode (`MODE_QUAD`) [S2, unofficial]. The face culling logic references Sebastian Aaltonen's approach [S2, unofficial].

## Sources
[S1] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S2] GitHub - clayjohn/voxel-renderer: A short experiment in high ..., https://github.com/clayjohn/voxel-renderer, fetched 2026-10-27 (unofficial)
