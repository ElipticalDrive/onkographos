# GPU Voxelization Using OpenGL Image Atomics

Also called: Conservative Rasterization Voxelization, OpenGL Image Atomic Voxelization, Scene Voxelization using GPU Rasterization

## Overview
GPU Voxelization Using OpenGL Image Atomics transforms continuous 3D polygonal meshes into a discrete, regular volumetric grid stored in GPU memory by projecting geometry along its dominant axis, rasterizing fragments without a traditional framebuffer, and utilizing fragment-stage image atomic operations to write directly into 3D textures [S2, unofficial]. It solves the problems of projection loss (degenerate triangles perpendicular to the view plane) and race conditions (concurrent writes to the same voxel) inherent in standard rasterization pipelines [S2, unofficial]. Unlike standard rasterization which projects onto a 2D viewport, this method decouples rasterization from the framebuffer, converting complex geometric meshes into solid or shell voxel structures in a single render pass [S2, unofficial]. It differs from [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md) by explicitly handling concurrency via hardware atomics and dominant-axis projection to ensure watertightness, and from [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md) by relying on rasterization primitives rather than explicit geometric intersection tests [G].

## How it works
**Inputs and Outputs**
*   **Input:** A polygonal mesh (vertices and indices) and a target 3D voxel grid resolution (typically $128^3$ or $256^3$) [S2, unofficial].
*   **Output:** A 3D texture (e.g., `GL_R32UI`) representing the voxel grid, where occupied voxels are marked via atomic operations [S2, unofficial].

**Data Structures**
*   **Voxel Grid:** A 3D texture with unsigned integer format `GL_R32UI` [S2, unofficial].
    *   Layout: 3D array of 32-bit unsigned integers.
    *   Binding: Bound as a coherent image using `glBindImageTexture` with `GL_READ_WRITE` access [S2, unofficial].
*   **Attribute Grids (Optional):** Separate 3D textures for accumulating attributes (e.g., color, normals) and a counter grid for averaging [S2, unofficial].

**Steps**
1.  **Geometry Shader Processing:**
    *   Calculate the triangle's face normal in the Geometry Shader using the cross product of two edges:
        $$ \text{normal} = \text{cross}(v_1 - v_0, v_2 - v_0) $$
    *   Compute the absolute value of the normal components:
        $$ \text{absNormal} = |\text{normal}| $$
    *   Determine the dominant projection plane by comparing the magnitudes of the normal components [S2, unofficial]:
        *   If $|N_z| \ge |N_x|$ and $|N_z| \ge |N_y|$, project onto the XY-plane.
        *   If $|N_x| \ge |N_y|$ and $|N_x| \ge |N_z|$, swizzle coordinates to project onto the YZ-plane.
        *   If $|N_y| \ge |N_x|$ and $|N_y| \ge |N_z|$, swizzle coordinates to project onto the ZX-plane.
    *   Apply an orthographic projection matrix matching the resolution of the target 3D voxel grid along the chosen axis [S2, unofficial].
    *   Pass the original world-space positions to the fragment shader as interpolants [S2, unofficial].
    *   **Conservative Rasterization:** Enable hardware conservative rasterization via `GL_NV_conservative_raster` (`glEnable(GL_CONSERVATIVE_RASTER_NV)`) or use a software fallback where the geometry shader expands the 2D projected triangle vertices outward along their edge normals by half a voxel's diagonal width [S2, unofficial].

2.  **Host Setup:**
    *   Disable color writes and depth tests:
        ```c
        glColorMask(GL_FALSE, GL_FALSE, GL_FALSE, GL_FALSE);
        glDisable(GL_DEPTH_TEST);
        glDisable(GL_CULL_FACE);
        ```
    *   Initialize the voxel grid storage:
        ```c
        glTexStorage3D(GL_TEXTURE_3D, 1, GL_R32UI, size, size, size);
        glBindImageTexture(0, voxelTexture, 0, GL_TRUE, 0, GL_READ_WRITE, GL_R32UI);
        ```
        [S2, unofficial]

3.  **Fragment Shader Execution:**
    *   Convert the interpolated world-space position into normalized coordinates within the voxel bounding box $[0, 1]$, and then scale to integer grid coordinates:
        $$ \text{voxelCoord} = \text{ivec3}(\text{normalizedWorldPos} \times \text{voxelGridResolution}) $$
    *   Declare the texture as a coherent image in the fragment shader:
        ```glsl
        layout(binding = 0, r32ui) uniform coherent uimage3D voxelGrid;
        ```
    *   Perform atomic writes based on the desired output type [S2, unofficial]:
        *   **Binary Occupancy:** Use `imageAtomicOr` to mark a voxel as occupied:
            ```glsl
            imageAtomicOr(voxelGrid, voxelCoord, 1u);
            ```
        *   **Averaging Attributes:** Use `imageAtomicAdd` to accumulate values alongside a separate counter buffer (since atomic floating-point addition is not universally supported):
            ```glsl
            imageAtomicAdd(countGrid, voxelCoord, 1u);
            imageAtomicAdd(colorGridR, voxelCoord, uint(color.r * 255.0));
            ```
        *   **Bit-Packed Composite Data:** Pack values into a single 32-bit unsigned integer using bit-shifts (e.g., `(R << 24) | (G << 16) | (B << 8) | A`) and write using `imageAtomicMax` to preserve dominant high-intensity colors or surface markers [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Host-side setup
def setup_voxelization(grid_size):
    voxel_texture = create_3d_texture(format=32, width=grid_size, height=grid_size, depth=grid_size)
    bind_image_texture(binding=0, texture=voxel_texture, access="READ_WRITE", format=32)
    gl_color_mask(False, False, False, False)
    gl_disable(GL_DEPTH_TEST)
    gl_disable(GL_CULL_FACE)
    if gl_supports_extension("GL_NV_conservative_raster"):
        gl_enable(GL_CONSERVATIVE_RASTER_NV)

# Geometry Shader Logic (Conceptual)
def geometry_shader_process(vertices):
    v0, v1, v2 = vertices
    normal = cross(v1 - v0, v2 - v0)
    abs_normal = abs(normal)
    if abs_normal.z >= abs_normal.x and abs_normal.z >= abs_normal.y:
        projection_plane = "XY"
    elif abs_normal.x >= abs_normal.y and abs_normal.x >= abs_normal.z:
        projection_plane = "YZ"
    else:
        projection_plane = "ZX"
    projected_vertices = apply_orthographic_projection(vertices, projection_plane, grid_size)
    if not conservative_rasterization_enabled:
        projected_vertices = expand_vertices_along_edge_normals(projected_vertices, voxel_diagonal / 2)
    return projected_vertices, vertices

# Fragment Shader Logic (Conceptual)
def fragment_shader_process(interpolated_world_pos):
    normalized_pos = map_to_bounding_box(interpolated_world_pos, min=0.0, max=1.0)
    voxel_coord = tuple(map(int, normalized_pos * grid_size))
    image_atomic_or(voxel_grid, voxel_coord, 1)
```

**Parameters**
*   `voxelGridResolution`: The resolution of the target 3D voxel grid, typically $128^3$ or $256^3$ [S2, unofficial].
*   `GL_R32UI`: The unsigned integer format used for the 3D texture backing the voxel grid [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Projection Loss:** A triangle perpendicular to the projection plane will degenerate into a line or disappear entirely, leaving holes in the voxel grid. This is solved by projecting along the dominant axis where the projected area is largest [S2, unofficial].
*   **Race Conditions:** Multiple fragments originating from different triangles, or overlapping segments of the same triangle, may map to the identical voxel cell simultaneously. This is solved by using OpenGL image atomic operations instead of standard `imageStore` [S2, unofficial].
*   **Missing Voxels:** Thin geometry or edges may cross a voxel's spatial bounds without covering the pixel center, resulting in missing voxels. This is solved by using conservative rasterization [S2, unofficial].

## Subtopics

### [GPU Voxelization with Atomic Accumulation](2559-gpu-voxelization-with-atomic-accumulation.md)
A fragment-shader-based voxelization technique that accumulates per-fragment lighting and material properties into a 3D texture using hardware atomic operations to resolve race conditions and compute a moving average of samples per voxel. Choose this when you need to store averaged attributes (like color or normal) rather than just binary occupancy.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Single-pass voxelization; handles concurrency via hardware atomics; ensures watertightness with conservative rasterization; decouples from framebuffer [S2, unofficial].
*   **Limitations:** Requires support for `GL_NV_conservative_raster` or software fallback; atomic operations may have performance overhead on some hardware; limited to integer formats for atomic operations (floating-point atomics not universally supported on images) [S2, unofficial].
*   **When to use:** When real-time voxelization of complex meshes is required and the target platform supports OpenGL image atomics and conservative rasterization [G].
*   **When not to use:** When floating-point atomic operations are strictly required and not supported; when the mesh is extremely sparse and a sparse voxel structure is more appropriate [G].

## Background
GPU Voxelization Using OpenGL Image Atomics is a technique developed to leverage modern OpenGL capabilities for real-time volumetric rendering [S2, unofficial]. It builds on the principles of rasterization-based voxelization, specifically addressing the limitations of standard rasterization pipelines when applied to 3D grids [S2, unofficial]. The method relies on the `GL_NV_conservative_raster` extension for hardware-accelerated conservative rasterization and OpenGL 4.2+ features for image atomics [S2, unofficial]. It is commonly used in real-time ray tracing, volumetric rendering, and collision detection systems where fast conversion of polygonal meshes to voxel grids is necessary [G].

## Sources
[S1] otaku690/SparseVoxelOctree, https://github.com/otaku690/SparseVoxelOctree, fetched 2026-10-26 (unofficial)
[S2] GPU Voxelization Using OpenGL Image Atomics - opengl, https://salivity.github.io/opengl/article/gpu-voxelization-using-open-gl-image-atomics, fetched 2026-10-29 (unofficial)
