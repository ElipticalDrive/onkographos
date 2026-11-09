# Cube Rasterization via SSBO

Also called: Voxel Instance Rendering, glDrawElementsInstanced Voxel Draw

## Overview
Renders $N$ voxels as instances of a base unit cube geometry using a single draw call, where each instance is transformed by an offset and colored via a palette texture lookup [S1, unofficial]. It rasterizes 3 faces of each cube per voxel to ensure correct visibility from the camera [S2, unofficial]. It differs from point-sprite methods by using explicit triangle geometry, providing exact cube geometry and explicit face culling logic at the cost of higher memory bandwidth [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of voxel positions and material properties; a camera view-projection matrix; a palette texture for albedo lookup.
*   **Output:** A rasterized image of voxel cubes.

**Data Structures:**
1.  **Index Buffer (EBO):** A static buffer containing 18 indices per voxel instance. The indices define 3 faces of a cube (6 triangles, 3 vertices per triangle). The indices are pre-computed with offsets to map to the 8 vertices of the base cube [S2, unofficial].
    *   Total indices: $N \times 18$, where $N$ is the number of instances [S2, unofficial].
    *   Index type: `GL_UNSIGNED_SHORT` [S1, unofficial].
2.  **Instance Data Buffer (SSBO):** A `GL_SHADER_STORAGE_BUFFER` bound at binding point 0. Contains one record per voxel [S2, unofficial].
    *   Layout: `uvec2 data[]` [S2, unofficial].
    *   `data.x`: Packs position. Bits [0:10] = X (11 bits), Bits [11:20] = Y (10 bits), Bits [21:31] = Z (11 bits) [S2, unofficial].
    *   `data.y`: Packs material properties. Bits [0:15] = ALBEDO (16 bits, R5G6B5 format), Bits [16:19] = METAL (4 bits), Bits [20:23] = ROUGH (4 bits), Bits [24:27] = INTENSITY (4 bits), Bits [28:31] = TYPE (4 bits) [S2, unofficial].
3.  **Vertex Buffer (VBO):** Contains the base cube geometry.
    *   Stride: 6 floats (24 bytes) per vertex [S1, unofficial].
    *   Layout: `[0:3]` `aLocalPosition` (vec3, float), `[3:6]` `aNormal` (vec3, float) [S1, unofficial].
4.  **Instance Attributes Buffer:** Contains per-instance offsets and UVs for palette lookup.
    *   Stride: 5 floats (20 bytes) per instance [S1, unofficial].
    *   Layout: `[0:3]` `aInstanceOffset` (vec3, float), `[3:5]` `aInstanceUV` (vec2, float) [S1, unofficial].
    *   Vertex attribute divisors for instance attributes are set to 1 [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:**
    *   Create and bind the static index buffer with 18 indices per instance. The indices for instance $i$ are derived from a base cube index array `cube_indices` such that `indices[i] = cube_indices[cube_local] + i * 8` [S2, unofficial].
    *   Populate the SSBO with voxel data packed into `uvec2` records [S2, unofficial].
    *   Create the VAO with the cube vertex attributes and instance attribute divisors [S1, unofficial].
2.  **Draw Call:**
    *   Bind the VAO, SSBO, and index buffer.
    *   Call `glDrawElementsInstanced(GL_TRIANGLES, N * 18, GL_UNSIGNED_SHORT, 0, N)` [S1, unofficial].
3.  **Vertex Shader Processing (per vertex):**
    *   Determine the instance index from `gl_VertexID`:
        $$ instance = \lfloor gl\_VertexID / 8 \rfloor $$
        (Implemented as `instance = gl_VertexID >> 3` in GLSL) [S2, unofficial].
    *   Fetch the instance data from the SSBO: `voxel_data = data[instance]` [S2, unofficial].
    *   Unpack position from `voxel_data.x` using bitmasks [S2, unofficial].
    *   Unpack albedo from `voxel_data.y`. Convert R5G6B5 to float3:
        $$ R = \frac{(albedo \gg 11) \& 31}{32.0}, \quad G = \frac{(albedo \gg 5) \& 63}{64.0}, \quad B = \frac{albedo \& 31}{32.0} $$
        (Note: Source [S2, unofficial] uses specific bit shifts for R5G6B5: `u_red = (u_albedo & 32) >> 5`, `u_green = (u_albedo & 960) >> 5`, `u_blue = (u_albedo & 63488) >> 11`? No, standard R5G6B5 is R:5, G:6, B:5. Source [S2, unofficial] code snippet shows: `u_red = (u_albedo & uint(32))>>uint(5)` which is bit 5, `u_green = (u_albedo & uint(960))>>uint(5)` bits 5-10, `u_blue = (u_albedo & uint(63488))>>uint(11)` bits 11-15. This implies a specific packing order or bit layout. We follow the source code exactly: `u_red = (u_albedo & 32) >> 5`, `u_green = (u_albedo & 960) >> 5`, `u_blue = (u_albedo & 63488) >> 11`) [S2, unofficial].
    *   Compute world position from the unpacked voxel coordinates [S2, unofficial].
    *   Determine the visible face by computing `local_camera_pos = camera_origin - world_position` [S2, unofficial].
    *   Determine the vertex's local face orientation (`xyz` bits from `gl_VertexID & 7`) and flip components based on the sign of `local_camera_pos` to ensure correct face culling [S2, unofficial].
    *   Transform the local cube vertex:
        $$ P_{local\_norm} = 2.0 \times uvw - 1.0 $$
        $$ P_{world\_final} = view\_projection\_matrix \times (P_{local\_norm} \times 0.05 + world\_position) $$
        (The factor 0.05 scales the unit cube) [S2, unofficial].
    *   Set `gl_Position` and output color [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic (GLSL-like)

# Constants
NUM_CUBE_VERTICES = 8
NUM_INDICES_PER_INSTANCE = 18

# Inputs
uniform uvec2 data[]; // SSBO
uniform mat4 view_projection_matrix;
uniform vec3 camera_origin;

in uint gl_VertexID;
in vec3 aLocalPosition;
in vec3 aNormal;
in vec3 aInstanceOffset;
in vec2 aInstanceUV;

out vec3 color;
out vec3 clip_pos;
out vec3 world_pos;

void main() {
    // 1. Determine instance index
    uint instance = gl_VertexID >> 3;
    
    // 2. Fetch instance data
    uvec2 voxel_data = data[instance];
    
    // 3. Unpack position (assuming 11:10:11 bit packing in x)
    // Note: Exact bit unpacking depends on the specific packing implementation in SSBO creation.
    // Assuming standard little-endian bit packing as implied by source [S2, unofficial].
    uint x = voxel_data.x & 0x7FF;          // 11 bits
    uint y = (voxel_data.x >> 11) & 0x3FF;  // 10 bits
    uint z = (voxel_data.x >> 21) & 0x7FF;  // 11 bits
    vec3 voxel_pos = vec3(float(x), float(y), float(z));
    
    // 4. Unpack Albedo (R5G6B5 from data.y)
    uint albedo = voxel_data.y;
    uint u_red = (albedo & uint(32)) >> uint(5);
    uint u_green = (albedo & uint(960)) >> uint(5);
    uint u_blue = (albedo & uint(63488)) >> uint(11);
    vec3 albedo_color = vec3(float(u_red) / 32.0, float(u_green) / 64.0, float(u_blue) / 32.0);
    
    // 5. Compute World Position
    // Note: Source [S2, unofficial] uses world_position derived from voxel_pos. 
    // Source [S1, unofficial] uses aInstanceOffset. We follow [S2, unofficial] for MODE_CUBE specifics.
    vec3 world_position = voxel_pos; 
    
    // 6. Determine Face Orientation
    vec3 local_camera_pos = camera_origin - world_position;
    uint vertex_bits = gl_VertexID & 0x7;
    uint x_bit = vertex_bits & 0x1;
    uint y_bit = (vertex_bits & 0x2) >> 1;
    uint z_bit = (vertex_bits & 0x4) >> 2;
    
    vec3 xyz = vec3(float(x_bit), float(y_bit), float(z_bit));
    
    // Flip based on camera relative position
    if (local_camera_pos.x > 0.0) {
        xyz.x = 1.0 - xyz.x;
    }
    if (local_camera_pos.y > 0.0) {
        xyz.y = 1.0 - xyz.y;
    }
    if (local_camera_pos.z > 0.0) {
        xyz.z = 1.0 - xyz.z;
    }
    
    // 7. Transform Vertex
    vec3 uvw = xyz;
    vec4 position = vec4(uvw * vec3(2.0) - vec3(1.0), 1.0);
    position = view_projection_matrix * vec4(position.xyz * 0.05 + world_position, 1.0);
    
    clip_pos = position.xyz;
    world_pos = world_position;
    gl_Position = position;
    color = albedo_color;
}
```

**Parameters:**
*   `num_cube_indices`: Total indices for all instances, calculated as $N \times 3 \times 2 \times 3$ (3 faces, 2 tris/face, 3 verts/tri) [S2, unofficial].
*   `MIN_ZOOM` and `MAX_ZOOM`: Clamping range for zoom factor, typically [0.3f, 3.0f] [S1, unofficial].
*   `ROTATION_SENSITIVITY`: Rotation degrees per pixel of drag, set to 0.4f [S1, unofficial].
*   `cameraDistance`: Base camera distance from model center, set to 5f [S1, unofficial].
*   `AMBIENT_INTENSITY`: 0.35f [S1, unofficial].
*   `SPECULAR_INTENSITY`: 0.5f [S1, unofficial].
*   `SHININESS`: 32f [S1, unofficial].
*   `LIGHT_POS_WORLD`: {10f, 15f, 10f} [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **VAO Availability:** VAOs are not universally available via pure GLES 3.0 on Android; explicit VBO/EBO binding may be required every frame, though overhead is negligible due to the single draw call [S1, unofficial].
*   **Zoom Artifacts:** The zoom factor must be clamped to prevent extreme scaling artifacts [S1, unofficial].
*   **Rotation Direction:** Rotation direction depends on drag direction across screen halves (right/left), handled naturally by the sign of `dx` [S1, unofficial].
*   **Precision:** Bit-packing position into `uvec2` limits coordinate precision (11:10:11 bits). Ensure voxel grid fits within these bounds [S2, unofficial].

## Subtopics

### [Procedural Instancing for Voxel Blocks](1637-procedural-instancing-for-voxel-blocks.md)
Generates voxel geometry procedurally in the vertex shader without an index buffer, using point sprites or instanced quads. Choose when memory bandwidth for index buffers is a bottleneck and exact cube geometry is less critical than fill rate.

### [Instanced Voxel Renderer](4102-instanced-voxel-renderer.md)
Uses hardware instancing with instance ID lookup to render voxels. Choose when leveraging standard GPU instancing features for simpler attribute management.

### [Hardware Instancing (Instance ID Lookup)](11005-hardware-instancing-instance-id-lookup.md)
Relies on `gl_InstanceID` to fetch data. Choose when the driver overhead of instance ID lookup is acceptable and simpler shader logic is preferred.

## Operating characteristics
*   **Time Complexity:** $O(1)$ draw call overhead plus $O(N)$ vertex processing for $N$ instances [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the SSBO and Index Buffer. The index buffer size is $18 \times N \times 2$ bytes (assuming 16-bit indices) [S2, unofficial].
*   **Measured Performance:** On Intel Integrated graphics, `MODE_CUBE` performs similarly to `MODE_QUAD` (point sprite) methods. On dedicated graphics cards, `MODE_QUAD` is approximately twice as fast as `MODE_CUBE` [S2, unofficial].
*   **Strengths:** Exact cube geometry; explicit face culling logic; single draw call for all voxels.
*   **Limitations:** Higher memory bandwidth usage due to index buffer and explicit triangle geometry; slower on dedicated GPUs compared to point sprite methods [S2, unofficial].
*   **When to Use:** When exact cube geometry and explicit face culling are required, or on hardware where dedicated GPU performance for point sprites is not significantly superior.
*   **When Not to Use:** When maximum fill rate is critical and the target hardware is a dedicated GPU where point sprite methods are significantly faster [S2, unofficial].

## Background
Originated from a technique described by Sebastian Aaltonen, implemented in the `voxel-renderer` project by Clay John [S2, unofficial]. It is also implemented in the `VoxelRenderer` class for Android OpenGL ES 3.0 visualization by Il-castor [S1, unofficial].

## Sources
[S1] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S2] GitHub - clayjohn/voxel-renderer: A short experiment in high ..., https://github.com/clayjohn/voxel-renderer, fetched 2026-10-27 (unofficial)
