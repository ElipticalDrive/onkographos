# Cube Rasterization via SSBO
Also called: GL_DRAW_ELEMENTS instancing, Instanced voxel rendering, Voxel Instance Rendering

## Overview
Cube Rasterization via SSBO is a voxel rendering technique that explicitly rasterizes the visible faces of each voxel as triangle geometry rather than using point sprites or ray-marching. It solves the problem of rendering large voxel fields with exact geometric boundaries and explicit face culling by packing voxel data into a Shader Storage Buffer Object (SSBO) and using a single large index buffer to draw 3 faces per voxel instance. It differs from [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md) by using explicit triangle primitives instead of implicit surface definitions, and from [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md) by relying on a pre-computed index buffer for face selection rather than procedural generation in the vertex shader.

## How it works
### Inputs and Outputs
- **Input**: A list of voxel instances, each defined by a position and material properties.
- **Output**: Rasterized triangles representing the visible faces of the voxels.

### Data Structures
1.  **Index Buffer**: A single static buffer containing indices for 3 faces of a cube. Each instance requires 18 indices (6 triangles). The total size is $N_{instances} \times 18$.
2.  **SSBO (Shader Storage Buffer Object)**: Bound at binding point 0. Contains one `uvec2` per voxel instance.
    -   `data.x`: Packed position.
        -   Bits 0-10: X coordinate (11 bits).
        -   Bits 11-20: Y coordinate (10 bits).
        -   Bits 21-31: Z coordinate (11 bits).
    -   `data.y`: Packed material properties.
        -   Bits 0-15: ALBEDO (16 bits, R5G6B5 format).
        -   Bits 16-19: METAL (4 bits).
        -   Bits 20-23: ROUGH (4 bits).
        -   Bits 24-27: INTENSITY (4 bits).
        -   Bits 28-31: TYPE (4 bits).

### Steps
1.  **Initialization**:
    -   Create a static index buffer defining 3 faces of a cube. The indices correspond to 6 triangles (18 indices total per instance).
    -   Create an SSBO large enough to hold `num_instances` `uvec2` elements.
    -   Populate the SSBO with voxel data, packing position and material properties as defined above.

2.  **Rendering Loop**:
    -   Bind the SSBO at binding point 0.
    -   Bind the static index buffer.
    -   Issue a draw call using `GL_DRAW_ELEMENTS` with instancing. The vertex count is 18 per instance.

3.  **Vertex Shader Processing**:
    -   Determine the instance index from `gl_VertexID`:
        $$ instance = \lfloor gl\_VertexID / 8 \rfloor $$
        [G] Note: The source code uses bit shift `gl_VertexID >> 3`, which is equivalent to integer division by 8.
    -   Retrieve voxel data from the SSBO using the instance index.
    -   Unpack position from `data.x` and albedo from `data.y`.
    -   Compute world position from the packed bits.
    -   Determine the local camera position:
        $$ local\_camera\_pos = camera\_origin - world\_position $$
    -   Determine the cube face orientation based on the sign of `local_camera_pos`. The vertex shader uses `gl_VertexID` to identify the local vertex within the cube (0-7). The face selection logic flips the vertex coordinates based on the camera direction to ensure only the 3 visible faces are rendered correctly.
        $$ xyz = uvec3(gl\_VertexID \& 0x1, (gl\_VertexID \& 0x2) >> 1, (gl\_VertexID \& 0x4) >> 2) $$
        For each axis $i \in \{x, y, z\}$:
        $$ if (local\_camera\_pos.i > 0.0) \quad xyz.i = 1 - xyz.i $$
    -   Transform the local vertex:
        $$ uvw = vec3(xyz) $$
        $$ position = view\_projection\_matrix \times vec4(uvw \times 2.0 - 1.0, 1.0) $$
        [G] The source code scales by 0.05 before adding world position. The formula in the source is:
        $$ gl\_Position = view\_projection\_matrix \times vec4(uvw \times 2.0 - 1.0, 1.0) \times 0.05 + world\_position $$
        Wait, the source code says: `position = view_projection_matrix * vec4(position.xyz * 0.05 + world_position, 1.0);`. This implies the scaling and translation happen in world space before projection.
        Correct interpretation:
        $$ local\_vertex = uvw \times 2.0 - 1.0 $$
        $$ world\_vertex = local\_vertex \times 0.05 + world\_position $$
        $$ gl\_Position = view\_projection\_matrix \times vec4(world\_vertex, 1.0) $$
    -   Output color and position.

### Parameters
-   `num_cube_indices`: Total number of indices in the index buffer.
    $$ num\_cube\_indices = num\_instances \times 3 \times 2 \times 3 $$
    (3 faces, 2 triangles per face, 3 vertices per triangle).

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic (GLSL-like)

# Inputs
uniform uvec2 voxel_data[MAX_INSTANCES];
uniform mat4 view_projection_matrix;
uniform vec3 camera_origin;

# Outputs
out vec3 color;
out vec3 world_pos;
out vec3 clip_pos;

# Constants
const float CUBE_SCALE = 0.05;

void main() {
    # Determine instance index
    uint instance_id = gl_VertexID >> 3;
    
    # Retrieve voxel data
    uvec2 data = voxel_data[instance_id];
    
    # Unpack position
    uint x = data.x & 0x7FF;       # 11 bits
    uint y = (data.x >> 11) & 0x3FF; # 10 bits
    uint z = (data.x >> 21) & 0x7FF; # 11 bits
    
    # Convert to world position (assuming normalized 0-1 range scaled by CUBE_SCALE)
    # Note: Exact mapping from bits to world coordinates depends on the specific implementation.
    # The source code implies a direct mapping, likely assuming a fixed grid or normalized coordinates.
    # For the purpose of this article, we assume the bits represent normalized coordinates [0, 1].
    vec3 world_position = vec3(float(x) / 2048.0, float(y) / 1024.0, float(z) / 2048.0);
    
    # Unpack albedo
    uint albedo = data.y & 0xFFFF;
    uint u_red = (albedo & 0x7C00) >> 11; # R5
    uint u_green = (albedo & 0x03E0) >> 5; # G6
    uint u_blue = albedo & 0x001F; # B5
    
    vec3 albedo_color = vec3(float(u_red) / 31.0, float(u_green) / 63.0, float(u_blue) / 31.0);
    
    # Determine face orientation
    vec3 local_camera_pos = camera_origin - world_position;
    uint vertex_local_id = gl_VertexID & 0x7; # 3 bits for 8 vertices
    uvec3 xyz = uvec3(vertex_local_id & 0x1, (vertex_local_id & 0x2) >> 1, (vertex_local_id & 0x4) >> 2);
    
    if (local_camera_pos.x > 0.0) {
        xyz.x = 1 - xyz.x;
    }
    if (local_camera_pos.y > 0.0) {
        xyz.y = 1 - xyz.y;
    }
    if (local_camera_pos.z > 0.0) {
        xyz.z = 1 - xyz.z;
    }
    
    vec3 uvw = vec3(xyz);
    
    # Transform vertex
    vec3 local_vertex = uvw * 2.0 - 1.0;
    vec3 world_vertex = local_vertex * CUBE_SCALE + world_position;
    
    gl_Position = view_projection_matrix * vec4(world_vertex, 1.0);
    
    color = albedo_color;
    world_pos = world_position;
    clip_pos = gl_Position.xyz;
}
```

### Edge Cases and Pitfalls
-   **Precision**: The position packing uses 11+10+11 bits. Ensure the world space coordinates fit within this range. If the voxel grid is larger, the precision may be insufficient, leading to visible artifacts.
-   **Face Culling**: The face selection logic relies on the camera position relative to the voxel. If the camera is exactly aligned with a voxel face, the sign check may be ambiguous. However, the strict inequality `> 0.0` ensures a consistent choice.
-   **Index Buffer Size**: The index buffer size grows linearly with the number of instances. For very large voxel counts, this may exceed GPU memory limits.

## Operating characteristics
-   **Time Complexity**: $O(N)$ where $N$ is the number of voxels. Each voxel requires a constant amount of work in the vertex shader.
-   **Memory Complexity**: $O(N)$ for the SSBO and index buffer. Each voxel uses 8 bytes in the SSBO and 18 indices in the index buffer.
-   **Performance**: On dedicated graphics cards, this method is approximately twice as slow as [Surface-based Instancing with Noise Modulation](1582-surface-based-instancing-with-noise-modulation.md) (MODE_QUAD). On Intel Integrated graphics, performance is similar.
-   **Strengths**: Exact cube geometry, explicit face culling, no ray-marching overhead in the fragment shader.
-   **Limitations**: Higher memory bandwidth usage due to explicit triangle geometry. Slower on dedicated GPUs compared to point-sprite based methods.

## Background
This method was described by Sebastian Aaltonen in a tweet [S1, unofficial]. It is implemented in the `voxel-renderer` project by Clay John [S1, unofficial]. The technique is part of a series of voxel rendering experiments exploring different rasterization strategies.

## Sources
[S1] GitHub - clayjohn/voxel-renderer: A short experiment in high ..., https://github.com/clayjohn/voxel-renderer, fetched 2026-10-27 (unofficial)
