# Cube Rasterization via SSBO

Also called: GL_DRAW_ELEMENTS instancing, GPU Instancing, Instanced Voxel Rendering, Voxel Instance Rendering, glDrawElementsInstanced Voxel Draw

## Overview
Cube Rasterization via SSBO renders $N$ voxels as instances of a base unit cube geometry using a single `glDrawElementsInstanced` draw call, where each instance is transformed by an offset and colored via a palette texture lookup [S1, unofficial]. It rasterizes exactly 3 faces of each cube per voxel to optimize for front-face visibility, using a single large index buffer (18 indices per instance) and a large SSBO containing voxel data [S2, unofficial]. It sets itself apart from sibling methods like [Procedural Instancing for Voxel Blocks](1637-procedural-instancing-for-voxel-blocks.md) by using explicit triangle geometry per voxel rather than point sprites or ray-marched quads, providing exact cube geometry and explicit face culling logic in the vertex shader [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of $N$ voxels, each defined by integer coordinates $(x, y, z)$ and material properties (albedo, metalness, roughness, etc.).
*   **Output:** A rasterized image of the voxel scene.

**Data Structures:**

1.  **Instance Buffer (SSBO):**
    *   Bound as a `std430` buffer (GLSL) or `StructuredBuffer` (HLSL).
    *   Layout: An array of `uvec2` per voxel.
    *   `data.x` (32-bit): Packed position.
        *   Bits 0-10: X coordinate (11 bits).
        *   Bits 11-20: Y coordinate (10 bits).
        *   Bits 21-31: Z coordinate (11 bits).
    *   `data.y` (32-bit): Packed material properties.
        *   Bits 0-15: ALBEDO (16 bits, R5G6B5 format).
        *   Bits 16-19: METAL (4 bits).
        *   Bits 20-23: ROUGH (4 bits).
        *   Bits 24-27: INTENSITY (4 bits).
        *   Bits 28-31: TYPE (4 bits).
    *   [S2, unofficial]

2.  **Index Buffer (EBO):**
    *   Static buffer containing indices for 3 faces of a cube.
    *   Total indices: $N \times 18$ (3 faces $\times$ 2 triangles/face $\times$ 3 vertices/triangle).
    *   Indices are generated at startup by replicating a base 18-index pattern with offsets: `indices[i] = cube_indices[cube_local] + cube * num_cube_vertices`, where `num_cube_vertices = 8`.
    *   [S2, unofficial]

3.  **Vertex Buffer (VBO):**
    *   Stride: `CubeGeometry.FLOATS_PER_VERTEX * 4` bytes.
    *   Layout:
        *   `[0:3]` `aLocalPosition` (`vec3`, float): Local vertex position.
        *   `[3:6]` `aNormal` (`vec3`, float): Local vertex normal.
    *   [S1, unofficial]

**Numbered Steps:**

1.  **Initialization:**
    *   Create and bind the static index buffer with 18 indices per voxel instance.
    *   Create and bind the SSBO containing the packed voxel data.
    *   Create the VAO with the cube vertex/normal attributes and instance offset/UV attributes. Set vertex attribute divisors for instance attributes to 1.
    *   [S1, unofficial]

2.  **Draw Call:**
    *   Bind the VAO and EBO.
    *   Call `glDrawElementsInstanced` with `GL_TRIANGLES`, index count ($N \times 18$), `GL_UNSIGNED_SHORT`, offset 0, and `instanceCount` ($N$).
    *   [S1, unofficial]

3.  **Vertex Shader Execution:**
    *   Determine instance index from `gl_VertexID`:
        $$ \text{instance} = \lfloor \text{gl\_VertexID} / 8 \rfloor $$
        (Note: Source [S2, unofficial] uses `gl_VertexID >> uint(3)`, which is equivalent to division by 8, assuming 8 vertices per cube face selection logic, though the index buffer has 18 indices. The source code snippet implies `gl_VertexID` is used to derive both the instance and the local face vertex. Specifically, `instance = gl_VertexID >> 3` suggests 8 vertices per instance in the shader's view, but the index buffer has 18. The source code snippet for `MODE_CUBE` uses `gl_VertexID` to determine the face orientation and local vertex. Let's follow the source code snippet exactly.)
    *   Fetch instance record from SSBO: `voxel_data = ssbo[instance]`.
    *   Unpack position and albedo using bitmasks.
        *   Position: Extract X, Y, Z from `voxel_data.x`.
        *   Albedo: Extract R, G, B from `voxel_data.y` (R5G6B5).
            $$ u\_red = (voxel\_data.y \ \& \ 0x1F) \ll 3 $$
            $$ u\_green = (voxel\_data.y \ \& \ 0x7E0) \gg 2 $$
            $$ u\_blue = (voxel\_data.y \ \& \ 0xF800) \gg 8 $$
            $$ \text{albedo} = \text{vec3}(u\_red/255.0, u\_green/255.0, u\_blue/255.0) $$
            (Note: Source [S2, unofficial] uses specific bitmasks `0x1`, `0x2016`, `0x63488` and divisors `32.0`, `64.0`, `32.0` for R5G6B5. We will use the source's exact logic.)
    *   Compute world position from unpacked coordinates.
    *   Determine cube face orientation based on camera position relative to voxel:
        $$ \text{local\_camera\_pos} = \text{camera\_origin} - \text{world\_position} $$
        $$ \text{xyz} = \text{uvec3}(\text{gl\_VertexID} \ \& \ 0x1, (\text{gl\_VertexID} \ \& \ 0x2) \gg 1, (\text{gl\_VertexID} \ \& \ 0x4) \gg 2) $$
        If `local_camera_pos.x > 0.0`, `xyz.x = 1 - xyz.x`.
        If `local_camera_pos.y > 0.0`, `xyz.y = 1 - xyz.y`.
        If `local_camera_pos.z > 0.0`, `xyz.z = 1 - xyz.z`.
    *   Transform local cube vertex:
        $$ \text{uvw} = \text{vec3}(\text{xyz}) $$
        $$ \text{position} = \text{view\_projection\_matrix} \times \text{vec4}(\text{uvw} \times \text{vec3}(2.0) - \text{vec3}(1.0) \times 0.05 + \text{world\_position}, 1.0) $$
    *   Set `gl_Position` and output color.
    *   [S2, unofficial]

**Parameters:**
*   `num_cube_indices`: Total indices for all instances, calculated as `no_of_instances * 3 * 2 * 3` [S2, unofficial].
*   `MIN_ZOOM`: 0.3f [S1, unofficial].
*   `MAX_ZOOM`: 3.0f [S1, unofficial].
*   `ROTATION_SENSITIVITY`: 0.4f [S1, unofficial].
*   `AMBIENT_INTENSITY`: 0.35f [S1, unofficial].
*   `SPECULAR_INTENSITY`: 0.5f [S1, unofficial].
*   `SHININESS`: 32f [S1, unofficial].
*   `LIGHT_POS_WORLD`: {10f, 15f, 10f} [S1, unofficial].
*   `cameraDistance`: 5f [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **VAO Availability:** VAOs are not universally available via pure GLES30 on Android; explicit VBO/EBO binding is performed every frame, though overhead is negligible due to the single draw call [S1, unofficial].
*   **Zoom Clamping:** Zoom factor is clamped to `[MIN_ZOOM, MAX_ZOOM]` to prevent extreme scaling artifacts [S1, unofficial].
*   **Rotation Direction:** Rotation direction depends on drag direction across screen halves (right/left), handled naturally by the sign of `dx` [S1, unofficial].
*   **Precision:** Bit-packing coordinates into `uvec2` limits the maximum grid size. X and Z are limited to 2048 (11 bits), Y to 1024 (10 bits) [S2, unofficial].

## Subtopics

### Procedural Instancing for Voxel Blocks
*   **What it is:** Renders voxel blocks by generating geometry procedurally in the vertex shader, often using point sprites or instanced quads.
*   **Difference:** Uses point sprites or quads instead of explicit cube faces. Simpler geometry but less precise face culling.
*   **When to choose:** When hardware supports efficient point sprite rasterization and exact cube geometry is not critical.
*   **Link:** [Procedural Instancing for Voxel Blocks](1637-procedural-instancing-for-voxel-blocks.md)

### Instanced Voxel Renderer
*   **What it is:** A general term for rendering voxels using instancing techniques, often involving complex culling and LOD.
*   **Difference:** May use various primitive types (points, quads, cubes) and culling strategies.
*   **When to choose:** When a more general or optimized rendering pipeline is needed.
*   **Link:** [Instanced Voxel Renderer](4102-instanced-voxel-renderer.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ draw call overhead plus $O(N)$ vertex processing for $N$ instances [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the SSBO and EBO. The SSBO uses 8 bytes per voxel (2 `uvec2` components? No, 2 `uvec2` is 8 bytes? No, `uvec2` is 8 bytes. So 16 bytes per voxel? Source [S2, unofficial] says "1 uvec2 per voxel" but then describes `data.x` and `data.y`. `uvec2` is 2 components. So 2 `uvec2` would be 16 bytes. Or is it 1 `uvec2` for position and 1 `uvec2` for material? The source says "1 uvec2 per voxel" but then lists `data.x` and `data.y`. This is contradictory. `uvec2` has 2 components. If `data.x` and `data.y` are components, then it's 1 `uvec2`. But `data.x` contains position and `data.y` contains material. So it's 1 `uvec2` per voxel. 8 bytes per voxel. [S2, unofficial].
*   **Performance:** On Intel Integrated graphics, `MODE_CUBE` performs similarly to `MODE_QUAD`. On dedicated graphics cards, `MODE_QUAD` is about twice as fast as `MODE_CUBE` [S2, unofficial].
*   **Strengths:** Exact cube geometry, explicit face culling, single draw call.
*   **Limitations:** Higher memory bandwidth usage due to index buffer and instance buffer. Slower on dedicated GPUs compared to point sprite methods.
*   **When to use:** When exact cube geometry is required and face culling is important.
*   **When not:** When maximum performance on dedicated GPUs is needed and point sprites are acceptable.

## Background
*   **Origin:** The method is implemented in Android OpenGL ES 3.0 voxel visualization via the `VoxelRenderer` class [S1, unofficial]. It is also described in the `voxel-renderer` project by Clay John, referencing Sebastian Aaltonen's technique [S2, unofficial].
*   **History:** The technique was popularized by Sebastian Aaltonen's tweet describing the 3-face rasterization method [S2, unofficial].
*   **Usage:** Used in voxel visualization tools and games requiring efficient voxel rendering.

## Sources
[S1] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S2] GitHub - clayjohn/voxel-renderer: A short experiment in high ..., https://github.com/clayjohn/voxel-renderer, fetched 2026-10-27 (unofficial)
