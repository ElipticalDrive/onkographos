# Orthographic Multi-View Voxelization

Also called: 3-Direction Rasterization, 3-View Voxelization, Directional Sheet Buffer Voxelization, Orthographic 3D Texture Rasterization, Voxelization via Orthographic Projection, Worksheet Voxelization

## Overview
Orthographic Multi-View Voxelization generates a 3D voxelized representation of scene geometry by rasterizing triangles into a 3D texture using three orthographic cameras aligned with the scene's bounding box axes [S1, unofficial]. It differs from [Single 2D Projection Voxelization](3482-single-2d-projection-voxelization.md) by using multiple views to ensure volumetric coverage, and from [3-Pass Voxelization](5300-3-pass-voxelization.md) by performing the aggregation directly into a 3D texture or via a 2D "worksheet" synthesis rather than separate depth-buffer passes [S1, S2, unofficial]. It solves the problem of efficiently converting complex triangle meshes into a dense 3D grid for subsequent volume rendering techniques like cone tracing [S1, unofficial].

## How it works
The method operates in two primary variants: direct 3D texture rasterization [S1, unofficial] and the 2D worksheet synthesis method [S2, unofficial]. Both rely on the parent's [Shared principles](5937-multi-view-projection.md) of camera configuration, rasterization, aggregation, and coordinate mapping.

### Inputs and Outputs
**Inputs:**
*   Triangle mesh geometry.
*   Scene Axis-Aligned Bounding Box (AABB) defined by `bb_min` and `bb_max` [S1, S2, unofficial].
*   Target voxel grid resolution $R = (R_x, R_y, R_z)$ [S1, unofficial].
*   Optional: Light data for pre-computation [S1, unofficial].

**Outputs:**
*   **Variant 1 (Direct):** A 3D texture (`texture3D`) of size $R_x \times R_y \times R_z$ containing RGBA32F voxel data [S1, unofficial].
*   **Variant 2 (Worksheet):** A 2D "worksheet" texture of resolution $2^W \times 2^H$ encoding the discrete voxel space $2^L \times 2^M \times 2^N$ such that $2^L \times 2^M \times 2^N = 2^W \times 2^H \times C$, where $C$ is the number of voxels encoded per texel (e.g., $C=32$ for binary voxelization with four 8-bit components) [S2, unofficial].

### Data Structures
1.  **3D Texture (Variant 1):**
    *   Layout: `std::vector<GLfloat>` of size $4 \times R_x \times R_y \times R_z$ [S1, unofficial].
    *   Initialization: All values set to `0.0f` (RGBA(0,0,0,0)) [S1, unofficial].
    *   Binding: Bound as a read-write image buffer (`GL_RGBA8` or `GL_RGBA32F`) for direct GPU write [S1, unofficial].

2.  **Sheet Buffers (Variant 2):**
    *   Three 2D buffers representing parts of the discrete voxel space, one for each axis direction (X, Y, Z) [S2, unofficial].
    *   These serve as intermediate storage during rasterization and texelization [S2, unofficial].

3.  **Worksheet (Variant 2):**
    *   Layout: 2D texture of resolution $2^W \times 2^H$ [S2, unofficial].
    *   Encoding: Each texel stores $C$ voxels. Access logic requires determining the patch by dividing the z-coordinate by $C$, accessing the texel at $(px, py)$, and using a lookup table to extract bits based on the offset within the component [S2, unofficial].

### Numbered Steps (Variant 1: Direct 3D Texture)
1.  **Calculate Bounding Box:** Compute `bb_min` and `bb_max` of the scene [S1, unofficial].
2.  **Compute Range and Scale:**
    *   $range = bb_{max} - bb_{min}$ [S1, unofficial].
    *   $scale = (R_x/range.x, R_y/range.y, R_z/range.z)$ [S1, unofficial].
3.  **Configure Cameras:**
    *   Create three orthographic cameras aligned with the X, Y, and Z axes [S1, unofficial].
    *   Set frustum bounds to $1.02 \times range$ on the viewing planes (passed as half-width `0.51f` to the projection matrix) [S1, unofficial].
    *   Set near plane to `0.1f` and far plane to $1.2 \times range + offset$ [S1, unofficial].
    *   Set camera offset to `0.2f` beyond the max bounds along each axis to ensure complete coverage [S1, unofficial].
4.  **Initialize Texture:** Clear the 3D texture to RGBA(0,0,0,0). Disable multisampling, depth testing, and face culling [S1, unofficial].
5.  **Rasterization Passes:**
    *   For each of the three views (X, Y, Z):
        *   Set the corresponding `viewProject` matrix uniform [S1, unofficial].
        *   Bind the 3D texture as a shader storage image [S1, unofficial].
        *   Upload `boxMin`, `scale`, and light data [S1, unofficial].
        *   Draw the scene geometry. The shader maps clip-space coordinates to voxel indices:
            $$ i = \text{floor}\left( \frac{p_{clip}.xyz - bb_{min}}{bb_{max} - bb_{min}} \times R \right) $$
            [S1, S2, unofficial].
        *   Fragments write directly to the 3D texture. Since depth testing is disabled, all intersecting fragments are written (atomic OR for occupancy) [S1, Parent, unofficial].
6.  **Generate Mipmaps:** Call `glGenerateTextureMipmap` on the filled 3D texture to enable multi-resolution cone tracing [S1, unofficial].

### Numbered Steps (Variant 2: Worksheet Synthesis)
1.  **Stage 1: Rasterization:**
    *   For each slab (or triangle), determine the axis direction (X, Y, or Z) along which the triangle has the maximum projection area [S2, unofficial].
    *   Rasterize the triangle against this axis direction into the corresponding directional sheet buffer [S2, unofficial].
    *   *Pitfall:* If a triangle is parallel to the rasterization direction, 2D linear interpolation results in a degenerate line segment. The solution is to always rasterize along the direction most parallel to the triangle's orientation [S2, unofficial].
2.  **Stage 2: Texelization:**
    *   Transform voxel positions to 3D volume coordinates immediately [S2, unofficial].
    *   Use these coordinates to find the correct position in the worksheet.
    *   Add voxels to the worksheet via a blending operation at the corresponding location [S2, unofficial].
3.  **Stage 3: Synthesis:**
    *   Transcode the three directional sheet buffers to the final worksheet [S2, unofficial].
    *   Transform each element to discrete voxel space, then encode to the appropriate texel in the worksheet [S2, unofficial].
    *   The worksheet reformulates the slabs of the volume along a desired axis direction [S2, unofficial].

### Pseudocode (Variant 1)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def orthographic_multi_view_voxelization(mesh, bb_min, bb_max, voxel_texture_size):
    """
    Variant 1: Direct 3D Texture Rasterization.
    """
    # 1. Calculate Range and Scale
    range_vec = bb_max - bb_min
    scale = (voxel_texture_size.x / range_vec.x,
             voxel_texture_size.y / range_vec.y,
             voxel_texture_size.z / range_vec.z)
    
    # 2. Initialize 3D Texture
    # Size: 4 * voxel_texture_size.x * voxel_texture_size.y * voxel_texture_size.z
    texture3D = [0.0] * (4 * voxel_texture_size.x * voxel_texture_size.y * voxel_texture_size.z)
    
    # 3. Configure Orthographic Cameras
    # Offset for camera extension
    offset = 0.2f
    # Frustum expansion factor
    projection_scale = 1.02f
    
    # Create 3 view-projection matrices for X, Y, Z axes
    # Note: Exact matrix construction depends on the graphics API, 
    # but must map scene AABB to clip space with the specified bounds.
    view_project_matrices = []
    for axis in [X_AXIS, Y_AXIS, Z_AXIS]:
        # Camera positioned along axis, looking at AABB
        # Frustum half-width = 0.51 * range (since 1.02 * range is full width)
        # Near = 0.1, Far = 1.2 * range + offset
        vp = create_orthographic_projection_matrix(
            half_width=0.51 * range_vec[axis],
            half_height=0.51 * range_vec[other_axis],
            near_plane=0.1f,
            far_plane=1.2 * range_vec[axis] + offset
        )
        view_project_matrices.append(vp)

    # 4. Rasterization Passes
    for i, vp in enumerate(view_project_matrices):
        # Bind texture3D as read-write image buffer
        bind_image_texture(texture3D, target=GL_RGBA8, access=GL_READ_WRITE)
        
        # Set shader uniforms
        set_uniform("viewProject[i]", vp)
        set_uniform("boxMin", bb_min)
        set_uniform("scale", scale)
        
        # Disable depth testing, face culling, multisampling
        disable(GL_DEPTH_TEST)
        disable(GL_CULL_FACE)
        disable(GL_MULTISAMPLE)
        
        # Draw mesh
        draw_mesh(mesh)

    # 5. Generate Mipmaps
    generate_mipmaps(texture3D)
    
    return texture3D
```

### Parameters
*   **voxelTextureSize:** Default resolution `glm::uvec3(64, 64, 64)` [S1, unofficial].
*   **offset:** Camera extension distance beyond the bounding box, fixed at `0.2f` [S1, unofficial].
*   **projection_scale:** Frustum expansion factor applied to range, fixed at `1.02f` (passed as `0.51f` to half-width) [S1, unofficial].
*   **C (Voxels per Texel):** Used in Variant 2. Example: $C=32$ for binary voxelization if a texel is composed of four 8-bit components [S2, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Rasterization:** If a triangle is parallel to the rasterization direction, 2D linear interpolation results in a line segment in the discrete voxel space. This is avoided by rasterizing along the direction most parallel to the triangle's orientation [S2, unofficial].
*   **Hardware Support:** Variant 1 relies on 3D texture support. At the time of the source's writing, this feature was not supported in consumer graphics hardware of the PC platform, making Variant 2 (2D worksheet) necessary for compatibility [S2, unofficial].
*   **Precision:** Coordinate mapping uses floor division. Ensure bounding box calculations are precise to avoid off-by-one errors at boundaries [S1, S2, unofficial].

## Subtopics

### [Shadow Volume Voxelization](5629-shadow-volume-voxelization.md)
Shadow Volume Voxelization generates a 3D shadow volume texture by voxelizing scene geometry into a 3D grid, where each voxel stores a bitmask representing which of its 8 constituent sub-voxels are occluded. It replaces standard occupancy flags with a bitmask per coarse voxel to enable multi-resolution shadow representation through mipmaps.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of triangles and $V$ is the number of voxels hit. In Variant 2, performance is significantly better than slicing-based methods. For a 256^3 resolution, Variant 2 takes ~100ms, whereas slicing-based voxelization takes ~5 seconds for the same model [S2, unofficial].
*   **Memory Complexity:** $O(R_x \cdot R_y \cdot R_z)$ for the 3D texture [S1, unofficial]. Variant 2 uses a 2D texture of size $2^W \times 2^H$ which encodes the 3D volume, reducing memory overhead by packing $C$ voxels per texel [S2, unofficial].
*   **Strengths:** Fast, hardware-accelerated, suitable for real-time applications. Generates mipmaps for multi-resolution rendering [S1, unofficial].
*   **Limitations:** Variant 1 requires 3D texture support which may be unavailable on older consumer hardware [S2, unofficial]. Variant 2 requires complex synthesis logic [S2, unofficial].
*   **When to use:** When real-time voxelization of complex models is required for volume rendering (e.g., cone tracing) [S1, S2, unofficial].
*   **When not to use:** When high-precision surface representation is needed without volumetric filling, or when hardware lacks 3D texture support and the overhead of 2D synthesis is unacceptable [S2, unofficial].

## Background
The method originates from research into real-time voxelization for complex models. The direct 3D texture approach is associated with Voxel Cone Tracing implementations [S1, unofficial]. The 2D worksheet synthesis method was proposed to overcome hardware limitations regarding 3D texture support and to provide a faster alternative to slicing-based voxelization [S2, unofficial]. It is used in applications requiring fast conversion of triangle meshes to 3D grids for volume rendering [S1, S2, unofficial].

## Sources
[S1] KyrieWei/VoxelConeTracing, https://github.com/KyrieWei/VoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
