# Naive Voxel Rendering

Also called: Cuboid-based Volume Rendering, Non-meshed voxel rendering, Per-voxel face rendering, Rasterization without LODs, Voxel Cuboid Rendering, VoxelShader::render

## Overview
Naive Voxel Rendering is an object-order technique that renders every visible voxel as an individual geometric primitive (typically a cube or cuboid) without merging adjacent faces into larger meshes or using Level of Detail (LOD) reduction [S1, S3, S4, unofficial]. It solves the problem of visualizing volumetric data by directly rasterizing the underlying voxel grid geometry [S7, unofficial]. Unlike [Base Renderer Geometry](3902-base-renderer-geometry.md) which may optimize mesh generation, this method explicitly avoids greedy meshing or face culling between adjacent voxels, drawing every face of every active voxel individually [S3, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid (volume data) where each cell contains a value (e.g., density, color, or binary existence) and grid coordinates $(x, y, z)$ [S3, S5, S7, unofficial].
*   **Output:** A 2D image rasterized from the camera's perspective, where each active voxel contributes its colored faces to the frame buffer [S4, S7, unofficial].

**Data Structures:**
The voxel data is typically stored in a 3D array or a packed 1D array representing the volume [S5, S7, unofficial].
*   **Voxel Data:** A structure or array holding the state of each voxel. In binary cases, a value of $0$ indicates empty, and $1$ (or $>0$) indicates presence [S3, S6, unofficial]. In continuous cases, it holds color or intensity values [S7, unofficial].
*   **Geometry Buffer:** Vertices are stored in an interleaved format or separate arrays. For a single cube, 6 faces are defined. Each face consists of 6 vertices (2 triangles) [S5, unofficial].
    *   **Vertex Layout:** Each vertex has position $(x, y, z)$ and texture coordinates $(u, v)$ [S2, unofficial].
    *   **Face Definition:** The 'front' face (Z+) vertices in local space are ordered as: $(-,-,+), (-,+,+), (+,+,+), (-,-,+), (+,-,+), (+,+,+)$ [S5, unofficial].
    *   **Normal Storage:** Normals are stored separately, with 3 floats per vertex, totaling 18 entries per face (6 vertices $\times$ 3 floats) [S5, unofficial].

**Parameters:**
*   $VoxelSize$: Physical size of a voxel in world units. Default is $1.0$ [S5, unofficial].
*   $VoxelHalfSize$: Half the voxel size, typically $0.5$ [S5, unofficial].
*   $CHUNK\_SIZE$: Width and depth dimension of a voxel chunk (e.g., 128) [S3, unofficial].
*   $CHUNK\_HEIGHT$: Height dimension of a voxel chunk (e.g., 32) [S3, unofficial].

**Algorithm Steps:**
1.  **Iterate:** Loop through all voxel coordinates $(x, y, z)$ within the defined chunk or world bounds [S3, S4, S7, unofficial].
2.  **Check Existence:** Determine if the voxel at $(x, y, z)$ is active (e.g., data value $> 0$ or $== 1$) [S3, S6, unofficial]. If inactive, skip or hide it (e.g., set Y to a large value) [S6, unofficial].
3.  **Frustum Culling (Optional):** Check if the voxel's position exceeds the camera frustum bounds. If so, return immediately to avoid processing [S2, unofficial].
4.  **Generate Geometry:** For each active voxel, generate the vertices for all 6 faces (or only visible faces if `cullInvisibleFaces` is enabled) [S2, S5, unofficial].
    *   Calculate the bottom-left corner position in view space: $P_{view} = P_{voxel} + \frac{1}{2}VoxelOffset + (-0.5, -0.5, 0)$ [S2, unofficial].
    *   Calculate other corners by adding offsets $(\pm 0.5, \pm 0.5, 0)$ [S2, unofficial].
    *   Transform positions using the Projection matrix $P$ and Model-View matrix $MV$ [S2, unofficial].
    *   If `cullInvisibleFaces` is enabled, calculate the dot product between the view-space normal and the vector to the camera. If $< 0$, the face is back-facing and is skipped [S2, unofficial].
5.  **Color Mapping:** Map the voxel's data value to a color. This can be a direct RGB value [S5, unofficial] or a Hue mapping based on intensity [S7, unofficial].
6.  **Render:** Issue a draw call for the voxel's geometry. This is often done via instancing or repeated calls to `glDrawArrays`/`glDrawElements` for each voxel [S3, S4, S6, unofficial].
    *   Set the model matrix to translate to $(x, y, z)$ [S3, unofficial].
    *   Bind the vertex array object (VAO) containing the cube geometry [S3, unofficial].
    *   Draw the triangles [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_naive_voxels(voxel_grid, chunk_size, chunk_height, voxel_size, projection_matrix, model_view_matrix):
    """
    Renders a voxel chunk using naive per-voxel face rendering.
    """
    # Pre-calculate half size
    half_size = voxel_size / 2.0
    
    # Iterate over the chunk dimensions
    for x in range(chunk_size):
        for y in range(chunk_height):
            for z in range(chunk_size):
                # Check if voxel exists
                if not voxel_grid.is_active(x, y, z):
                    continue
                
                # Optional Frustum Culling
                world_pos = voxel_grid.get_world_position(x, y, z)
                if not in_frustum(world_pos):
                    continue

                # Get voxel color
                color = voxel_grid.get_color(x, y, z)
                
                # Generate and draw 6 faces
                for face in range(6):
                    # Optional: Back-face culling
                    if cull_invisible_faces:
                        normal = get_face_normal(face)
                        view_normal = transform_vector(normal, model_view_matrix)
                        cam_dir = normalize(camera_position - world_pos)
                        if dot(view_normal, cam_dir) < 0:
                            continue

                    # Generate vertices for this face
                    vertices = generate_cube_face_vertices(x, y, z, face, voxel_size)
                    
                    # Transform vertices to clip space
                    clip_vertices = []
                    for v in vertices:
                        # Transform position
                        pos = transform_point(v, model_view_matrix)
                        pos = transform_point(pos, projection_matrix)
                        clip_vertices.append(pos)
                    
                    # Issue draw call for this face (2 triangles)
                    # In a true naive implementation, this might be a separate draw call
                    # or part of a larger instanced batch. Here we assume a generic draw function.
                    draw_triangles(clip_vertices, color)

def generate_cube_face_vertices(x, y, z, face, voxel_size):
    """
    Returns 6 vertices (2 triangles) for a specific face of a voxel at grid coords (x,y,z).
    Local offsets are applied relative to the voxel center.
    """
    # Base position of the voxel center in world space
    center_x = x * voxel_size
    center_y = y * voxel_size
    center_z = z * voxel_size
    
    # Define the 6 vertices for a unit cube centered at 0,0,0
    # Face 0: Front (Z+), Face 1: Back (Z-), etc.
    # Simplified: Return local coordinates scaled by voxel_size
    offsets = get_face_offsets(face)
    vertices = []
    for dx, dy, dz in offsets:
        vx = center_x + dx * voxel_size
        vy = center_y + dy * voxel_size
        vz = center_z + dz * voxel_size
        vertices.append((vx, vy, vz))
    return vertices

def get_face_offsets(face):
    """
    Returns the 6 vertex offsets for a face.
    Example for Front Face (Z+):
    (-0.5, -0.5, 0.5), (-0.5, 0.5, 0.5), (0.5, 0.5, 0.5),
    (-0.5, -0.5, 0.5), (0.5, -0.5, 0.5), (0.5, 0.5, 0.5)
    """
    # This is a simplified representation. Actual implementation depends on face index.
    # For brevity, returning a placeholder structure that implies the geometry.
    # In a real implementation, this would return specific tuples for each of the 6 faces.
    if face == 0: # Front
        return [(-0.5, -0.5, 0.5), (-0.5, 0.5, 0.5), (0.5, 0.5, 0.5),
                (-0.5, -0.5, 0.5), (0.5, -0.5, 0.5), (0.5, 0.5, 0.5)]
    # ... other faces ...
    return []
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating point errors in position calculation can cause z-fighting if voxels are extremely small or the camera is very close [G].
*   **Boundaries:** Ensure loops respect the chunk boundaries ($0$ to $CHUNK\_SIZE-1$) to avoid out-of-bounds memory access [S3, unofficial].
*   **Degenerate Input:** Empty chunks should be skipped entirely to avoid unnecessary draw calls [S3, unofficial].

## Subtopics

### [Per-voxel Projection (Object-order)](1816-per-voxel-projection-object-order.md)
Optimizes the naive approach for Maximum Intensity Projection (MIP) by sorting voxels by value, allowing simple overwriting of pixel values instead of standard alpha blending or depth testing.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, multiplied by a constant factor for the number of faces (6) and vertices per face (6). Specifically, $36 \times N$ vertices are processed [S4, S5, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the voxel data and temporary geometry buffers.
*   **Performance:** Extremely inefficient for large volumes. A $128 \times 32 \times 32$ chunk generates 786,432 draw calls if not instanced [S3, unofficial]. Performance scales linearly with voxel count but degrades rapidly due to CPU-GPU overhead of many draw calls [S4, S7, unofficial].
*   **Strengths:** Simple to implement; no complex meshing algorithms required; easy to debug individual voxels [S5, unofficial].
*   **Limitations:** High draw call overhead; no face culling between adjacent voxels (every voxel draws all 6 faces unless explicitly checked); memory intensive for large datasets [S5, S7, unofficial].
*   **When to use:** Small volumes, debugging, or educational purposes where simplicity outweighs performance [S5, unofficial].
*   **When not to use:** Large-scale voxel engines or real-time applications requiring high frame rates [S4, S7, unofficial].

## Background
Naive Voxel Rendering is a foundational technique in voxel graphics, often used as a baseline for comparison against optimized methods like greedy meshing or ray marching [S1, S3, unofficial]. It appears in early voxel engine tutorials and simple demonstration projects [S4, S5, unofficial]. The method is cited in contexts discussing the performance pitfalls of non-optimized voxel rendering, such as Mathematica demonstrations or basic OpenGL tutorials [S3, S7, unofficial].

## Sources
[S1] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S2] mmalesadev/svo_renderer, https://github.com/mmalesadev/svo_renderer, fetched 2026-10-26 (unofficial)
[S3] OpenGL Meshing Slower Than Not Meshing, https://stackoverflow.com/questions/66466017/opengl-meshing-slower-than-not-meshing, fetched 2026-10-26 (unofficial)
[S4] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S5] JamesRandall/SimpleVoxelEngine, https://github.com/JamesRandall/SimpleVoxelEngine, fetched 2026-10-26 (unofficial)
[S6] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S7] How to show solid bodies using volumetric rendering?, https://mathematica.stackexchange.com/questions/6247, fetched 2026-10-30 (unofficial)
