# Orthographic Multi-View Voxelization

Also called: 3-Direction Rasterization, 3-View Voxelization, Directional Sheet Buffer Voxelization, Orthographic 3D Texture Rasterization, Voxelization via Orthographic Projection, Worksheet Voxelization

## Overview
Orthographic Multi-View Voxelization rasterizes scene geometry into a 3D texture or a compressed 2D "worksheet" representation using three orthographic cameras aligned with the principal axes of the scene's bounding box [S1, S2, unofficial]. It solves the problem of generating a volumetric representation for subsequent volume rendering (such as cone tracing) by ensuring every surface element is visible from at least one viewpoint, thereby preventing "shadow" voxels caused by occlusion or grazing angles [S1, S2, unofficial]. It differs from [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md) by using three views to cover all surface normals, and from [3-Pass Voxelization](5300-3-pass-voxelization.md) by typically performing the synthesis in a single GPU-driven pass or via efficient 2D-to-3D transcoding rather than explicit CPU-side merging of three separate 3D volumes [S2, unofficial].

## How it works
The method operates in two primary variations: direct 3D texture rasterization [S1, unofficial] and the sheet-buffer/worksheet synthesis approach [S2, unofficial]. Both rely on the parent's [Shared principles](5937-multi-view-projection.md).

### Inputs and Outputs
*   **Inputs:** Scene geometry (triangles), scene bounding box ($bb_{min}, bb_{max}$), target voxel grid resolution ($V_x, V_y, V_z$).
*   **Outputs:** A 3D texture (RGBA32F) representing the voxel grid [S1, unofficial], or a 2D "worksheet" texture encoding the discrete voxel space [S2, unofficial].

### Data Structures

**1. Direct 3D Texture Approach [S1, unofficial]:**
*   `texture3D`: A `std::vector<GLfloat>` or GPU image buffer of size $4 \times V_x \times V_y \times V_z$. Initialized to `0.0f`. Represents RGBA32F voxel data.
*   `voxelTextureSize`: `glm::uvec3(64, 64, 64)` by default [S1, unofficial].

**2. Worksheet Approach [S2, unofficial]:**
*   **Sheet Buffers:** Three 2D buffers representing parts of the discrete voxel space, one for each axis direction (X, Y, Z).
*   **Worksheet:** A 2D texture of resolution $2^W \times 2^H$. It encodes the discrete voxel space ($2^L \times 2^M \times 2^N$) such that:
    $$ 2^L \times 2^M \times 2^N = 2^W \times 2^H \times C $$
    where $C$ is the number of voxels encoded in a single texel (e.g., $C=32$ for binary voxelization with four 8-bit components) [S2, unofficial].
*   **Voxel Access Logic:** To access a voxel at discrete coordinates $(p_x, p_y, p_z)$:
    1.  Determine the patch index by dividing $p_z$ by $C$: $patch\_idx = \lfloor p_z / C \rfloor$.
    2.  Access the texel at $(p_x, p_y)$ within the worksheet.
    3.  Determine the bit offset within the texel components using $p_z \pmod C$ and $C$.
    4.  Use a lookup table to extract the specific bits representing the voxel state [S2, unofficial].

### Steps

#### Variation A: Direct 3D Texture Rasterization [S1, unofficial]

1.  **Initialization:**
    *   Calculate scene bounding box min ($bb_{min}$) and max ($bb_{max}$).
    *   Compute range: $range = bb_{max} - bb_{min}$.
    *   Set camera offset: $offset = 0.2f$ [S1, unofficial].
    *   Clear the 3D texture to RGBA(0,0,0,0).
    *   Disable multisampling, depth testing, and face culling.
    *   Bind the 3D texture as a read-write image buffer (e.g., `GL_RGBA8` or `GL_RGBA32F`) [S1, unofficial].

2.  **Camera Setup:**
    *   Position three orthographic cameras along the X, Y, and Z axes.
    *   Set camera offset to $0.2f$ beyond the max bounds along each axis [S1, unofficial].
    *   Set orthographic projection frustum bounds to $1.02 \times range$ on the viewing planes.
        *   Near plane: $0.1f$.
        *   Far plane: $1.2 \times range + offset$ [S1, unofficial].
    *   Generate view-projection matrices for each axis. Pass them to shader uniforms `viewProject[0]`, `viewProject[1]`, and `viewProject[2]` [S1, unofficial].

3.  **Shader Uniforms:**
    *   `boxMin`: Set to $bb_{min}$ [S1, unofficial].
    *   `scale`: Set to $(V_x/range.x, V_y/range.y, V_z/range.z)$ [S1, unofficial].
    *   `projection_scale`: Frustum expansion factor, fixed at $1.02f$ (passed as $0.51f$ for half-width) [S1, unofficial].

4.  **Rasterization Pass:**
    *   Upload light data and scene geometry.
    *   Draw the scene geometry three times (once for each view-projection matrix).
    *   In the voxelization shader, transform fragment positions from clip space to voxel grid space using the `scale` and `boxMin` uniforms.
    *   Perform atomic writes to the 3D texture image buffer at the calculated voxel coordinates [S1, unofficial].

5.  **Post-Processing:**
    *   Generate mipmaps for the filled 3D texture to enable multi-resolution cone tracing [S1, unofficial].

#### Variation B: Sheet Buffer and Worksheet Synthesis [S2, unofficial]

1.  **Stage 1: Rasterization (Sheet Buffers):**
    *   For each of the three axes (X, Y, Z), create a directional sheet buffer.
    *   Identify triangles intersecting the current slab/direction.
    *   **Degeneracy Handling:** Determine the rasterization direction for each triangle. If a triangle is parallel to a view direction, 2D linear interpolation results in a line segment. Rasterize the triangle along the axis direction where it has the **maximum projection area** (most parallel to its normal) [S2, unofficial].
    *   Rasterize the triangle onto the corresponding sheet buffer.

2.  **Stage 2: Texelization (Transformation):**
    *   Transform the rasterized voxel positions to 3D volume coordinates immediately.
    *   Use these coordinates to find the correct position in the intermediate worksheet structure.
    *   Add voxels to the worksheet via a blending operation at the corresponding location [S2, unofficial].

3.  **Stage 3: Synthesis (Transcoding):**
    *   Transcode the three directional sheet buffers into the final 2D worksheet texture.
    *   Transform each element from the sheet buffers to discrete voxel space.
    *   Encode the discrete voxel data into the appropriate texel in the worksheet according to the formula $2^L \times 2^M \times 2^N = 2^W \times 2^H \times C$ [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Variation A: Direct 3D Texture Rasterization [S1, unofficial]

def orthographic_voxelization_direct(scene, voxel_texture_size):
    # 1. Initialization
    bb_min, bb_max = scene.calculate_bounding_box()
    range_vec = bb_max - bb_min
    offset = 0.2f
    
    # Clear 3D texture
    texture = create_3d_texture(voxel_texture_size, format=RGBA32F, clear_value=0.0f)
    
    # Disable depth/culling for write-through
    gl_disable(GL_DEPTH_TEST)
    gl_disable(GL_CULL_FACE)
    gl_disable(GL_MULTISAMPLE)
    
    # Bind texture as image buffer
    bind_image_texture(0, texture, access=GL_READ_WRITE, format=GL_RGBA8)
    
    # 2. Camera Setup
    # Three views: X, Y, Z
    view_projections = []
    for axis in [X_AXIS, Y_AXIS, Z_AXIS]:
        # Camera position offset beyond max bounds
        cam_pos = bb_max + (axis * offset)
        
        # Frustum bounds: 1.02x range
        half_width = 0.51f * range_vec[axis_perp] # Perpendicular axes
        near_plane = 0.1f
        far_plane = 1.2f * range_vec[axis] + offset
        
        vp = create_orthographic_projection(
            left=-half_width, right=half_width,
            bottom=-half_width, top=half_width,
            near=near_plane, far=far_plane
        )
        view = create_look_at(cam_pos, bb_min, axis_up(axis))
        view_projections.append(view * vp)
        
    # 3. Shader Uniforms
    scale = voxel_texture_size / range_vec
    set_uniform("boxMin", bb_min)
    set_uniform("scale", scale)
    set_uniform("projection_scale", 0.51f) # Half-width factor
    
    # 4. Rasterization Pass
    for vp in view_projections:
        set_uniform("viewProject", vp)
        draw_scene(scene)
        
    # 5. Post-Processing
    generate_mipmaps(texture)
    return texture

# Variation B: Worksheet Synthesis [S2, unofficial]

def orthographic_voxelization_worksheet(scene, resolution):
    # Resolution: 2^L x 2^M x 2^N
    # Worksheet: 2^W x 2^H
    # C: Voxels per texel
    
    sheet_buffers = [create_2d_buffer() for _ in range(3)] # X, Y, Z sheets
    
    # Stage 1: Rasterization
    for axis_idx, axis in enumerate([X_AXIS, Y_AXIS, Z_AXIS]):
        sheet = sheet_buffers[axis_idx]
        
        for triangle in scene.triangles:
            # Check intersection with slab
            if not intersects_slab(triangle, axis):
                continue
                
            # Degeneracy handling: Choose direction with max projection area
            proj_area_x = abs(dot(triangle.normal, X_AXIS))
            proj_area_y = abs(dot(triangle.normal, Y_AXIS))
            proj_area_z = abs(dot(triangle.normal, Z_AXIS))
            
            raster_axis = argmax([proj_area_x, proj_area_y, proj_area_z])
            
            # Rasterize triangle onto sheet buffer along raster_axis
            rasterize_triangle_to_sheet(triangle, sheet, raster_axis)
            
    # Stage 2 & 3: Synthesis
    worksheet = create_2d_texture(width=2^W, height=2^H)
    
    for axis_idx, axis in enumerate([X_AXIS, Y_AXIS, Z_AXIS]):
        sheet = sheet_buffers[axis_idx]
        
        for voxel_pos in sheet.get_voxels():
            # Transform to discrete voxel space
            px, py, pz = transform_to_discrete(voxel_pos, axis)
            
            # Encode into worksheet
            # Determine patch
            patch_idx = floor(pz / C)
            
            # Access texel at (px, py) in patch
            texel = worksheet.get_texel(px, py, patch_idx)
            
            # Determine bit offset
            bit_offset = (pz % C) * bits_per_voxel
            
            # Update texel bits
            worksheet.set_texel_bits(px, py, patch_idx, bit_offset, voxel_state)
            
    return worksheet
```

### Parameters
*   `voxelTextureSize`: Default resolution `glm::uvec3(64, 64, 64)` [S1, unofficial].
*   `offset`: Camera extension distance, fixed at `0.2f` [S1, unofficial].
*   `projection_scale`: Frustum expansion factor, fixed at `1.02f` (passed as `0.51f` to half-width) [S1, unofficial].
*   `C` (Voxels per Texel): Number of voxels encoded in a single texel. Example: $C=32$ for binary voxelization if a texel is composed of four 8-bit components [S2, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Rasterization:** If a triangle is parallel to the rasterization direction, 2D linear interpolation results in a line segment in the discrete voxel space. This is handled by rasterizing along the direction most parallel to the triangle's orientation (maximum projection area) [S2, unofficial].
*   **Hardware Support:** Rendering directly to 3D textures was not supported in consumer graphics hardware of PC platforms at the time of the worksheet method's publication [S2, unofficial].

## Subtopics

### Slicing-based voxelization
A method that rasterizes the model the same number of times as the resolution of the volume along the z-direction [S2, unofficial]. It is significantly slower than the proposed 2D texture method; for the Blade model at $256^3$, it takes ~5 seconds compared to ~100ms for the proposed method [S2, unofficial]. This feature (rendering to 3D textures) has not been supported in consumer graphics hardware of PC platform at the time of writing [S2, unofficial].

### [Shadow Volume Voxelization](5629-shadow-volume-voxelization.md)
Shadow Volume Voxelization generates a 3D shadow volume texture by voxelizing scene geometry into a 3D grid, where each voxel stores a bitmask representing which of its 8 constituent sub-voxels are occluded. It replaces standard occupancy flags with a bitmask per coarse voxel to enable multi-resolution shadow representation through mipmaps.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$ where $N$ is the number of triangles and $V$ is the number of views (3). The worksheet synthesis adds a transcoding step proportional to the number of voxels/texels.
*   **Memory Complexity:** $O(V_x \cdot V_y \cdot V_z)$ for the 3D texture approach [S1, unofficial]. The worksheet approach uses $O(2^W \cdot 2^H)$ where $2^W \cdot 2^H \cdot C = V_x \cdot V_y \cdot V_z$, offering compression [S2, unofficial].
*   **Measured Performance:** For the Blade model at $256^3$ resolution, the worksheet method takes ~100ms, whereas slicing-based voxelization takes ~5 seconds [S2, unofficial].
*   **Strengths:** Fast, hardware-accelerated, handles complex models efficiently, avoids CPU-side depth sorting [S2, unofficial].
*   **Limitations:** Requires atomic writes or complex synthesis logic; direct 3D texture support may be limited on older hardware [S2, unofficial].

## Background
The method originates from research into real-time voxelization for complex models, specifically the "Worksheet Voxelization" technique presented in *Real-time Voxelization for Complex Models* [S2, unofficial]. It is also implemented in the *Voxel Cone Tracing* demo by KyrieWei [S1, unofficial]. The approach leverages standard graphics hardware rasterization pipelines to bypass the limitations of CPU-based voxelization and the performance costs of slicing-based methods.

## Sources
[S1] KyrieWei/VoxelConeTracing, https://github.com/KyrieWei/VoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
