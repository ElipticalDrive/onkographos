# View-Aligned Slicing

Also called: View-Aligned Proxy Geometry Sampling, View-Aligned Slice Rendering

## Overview
View-Aligned Slicing is a direct volume rendering technique that decomposes a 3D volume into a set of 2D planar slices (proxy geometry) which are dynamically rotated to align with the camera's view plane [S2, unofficial][S3, unofficial]. Unlike standard ray casting which samples the volume texture directly per pixel, this method rasterizes explicit geometric primitives (triangles or quads) representing the slices, requiring geometry regeneration when the view changes [S2, unofficial]. It sets itself apart from siblings like [Hybrid Ray Tracing](2308-hybrid-ray-tracing.md) by leveraging GPU rasterization for compositing, and from [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md) by operating on scalar field data rather than extracted isosurfaces.

## How it works
**Inputs and Outputs:**
- **Input:** 3D volumetric data (scalar field), camera transform (view and projection matrices), number of slices ($N$).
- **Output:** 2D image with alpha-blended volume visualization.

**Data Structures:**
1.  **3D Texture:** Stores the volumetric data. Clamping mode must be set to `GL_CLAMP_TO_BORDER` to avoid artifacts during texture coordinate transformation [S3, unofficial].
2.  **Vertex Buffer (`a_position`):** A `Float32Array` containing flat vertex coordinates $[x, y, z]$ for all slices concatenated together [S2, unofficial].
3.  **Offsets Array:** An array of tuples `[offset, length]` where `offset` is the byte offset in the vertex buffer and `length` is the number of vertices for that specific slice [S2, unofficial].
4.  **Shader Uniforms:** Per-slice or global uniforms including `sliceNumber`, `totalSlices`, `MVPMatrix`, `ModelMatrix`, `ViewMatrix`, and `TextureMatrix` [S3, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    - Load 3D volume data into a 3D texture [S3, unofficial].
    - Scale a unit cube (or quad) with the volume's physical dimensions [S3, unofficial].
    - Determine the number of slices. Initialization may generate $2N$ slices for buffering, while the draw pass uses $N$ slices [S2, unofficial].

2.  **Geometry Generation (Per Frame):**
    - Intersect slicing planes with the volume bounding box to define the polygonal shape of each slice [S1, unofficial][S2, unofficial].
    - Calculate view-aligned geometry by applying a rotation matrix derived from the camera's `lookAtMatrix` and `translationMatrix` [S2, unofficial]. This rotation aligns the slices with the view plane (typically the X-Y plane) [S3, unofficial].
    - *Alternative Implementation:* Keep the slice geometry (unit quad) stationary and transform the texture coordinates in the vertex shader using a rotation matrix [S3, unofficial].
    - Sort the resulting vertices in counter-clockwise order [S1, unofficial].
    - Construct polygon primitives from the centroid as a triangle fan [S1, unofficial].
    - Update the vertex attribute buffer `a_position` with the newly generated slice data [S2, unofficial].

3.  **Rendering Loop:**
    - Enable blending: `gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)` [S2, unofficial].
    - Disable face culling during rendering to handle arbitrary slice orientations, then re-enable if needed [S2, unofficial].
    - Bind the vertex buffer and enable the vertex attribute array for `a_position` [S2, unofficial].
    - Iterate through the `offsets` array. For each slice:
        - Set the vertex pointer: size 3, type `gl.FLOAT`, normalized false, stride 0, offset calculated as `offset * 4` (for `Float32` byte alignment) [S2, unofficial].
        - Draw the slice using `gl.drawArrays(gl.TRIANGLES, 0, length)` [S2, unofficial].
    - The fragment shader samples the 3D texture at the interpolated texture coordinates to retrieve the scalar value [S1, unofficial][S2, unofficial].
    - Apply the transfer function (e.g., cubic spline fit on manual points) to map scalar values to color and opacity [S3, unofficial].
    - Composite fragments using alpha blending [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_view_aligned_slices(number_of_slices, look_at_matrix, translation_matrix):
    """
    Generates vertex data for view-aligned slices.
    Returns: (vertices_array, offsets_length_array)
    """
    vertices = []
    offsets_length = []
    
    # Calculate rotation to align slices with view plane
    # Note: Exact rotation matrix derivation depends on camera orientation
    rotation_matrix = compute_rotation_from_look_at(look_at_matrix)
    
    for i in range(number_of_slices):
        # Determine slice position along the view-aligned axis
        # Assuming slices are stacked along the negative Z direction in view space
        slice_z = -i * slice_spacing
        
        # Generate vertices for the slice polygon
        # Method A: Rotate geometry
        slice_vertices = generate_slice_polygon(slice_z, rotation_matrix)
        
        # Method B (Alternative): Keep geometry static, transform texture coords
        # slice_vertices = generate_unit_quad()
        
        # Store vertices
        start_index = len(vertices)
        vertices.extend(slice_vertices)
        
        # Record offset (in bytes) and length (in vertices)
        byte_offset = start_index * 4  # 4 bytes per float
        length = len(slice_vertices)
        offsets_length.append([byte_offset, length])
        
    return np.array(vertices, dtype=np.float32), offsets_length

def render_view_aligned_slices(program, volume_texture, transfer_fn_texture, vertices, offsets_length, mvp_matrix):
    """
    Renders the view-aligned slices.
    """
    gl.enable(gl.BLEND)
    gl.blend_func(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)
    gl.disable(gl.CULL_FACE)
    
    gl.use_program(program)
    
    # Bind textures
    gl.active_texture(gl.TEXTURE0)
    gl.bind_texture(gl.TEXTURE_3D, volume_texture)
    gl.uniform1i(gl.get_uniform_location(program, "volume"), 0)
    
    gl.active_texture(gl.TEXTURE1)
    gl.bind_texture(gl.TEXTURE_2D, transfer_fn_texture)
    gl.uniform1i(gl.get_uniform_location(program, "transfer_fn"), 1)
    
    # Set MVP Matrix
    gl.uniform_matrix4fv(gl.get_uniform_location(program, "MVPMatrix"), mvp_matrix)
    
    # Bind vertex buffer
    vertex_buffer = create_buffer(vertices)
    gl.bind_buffer(gl.ARRAY_BUFFER, vertex_buffer)
    
    # Enable vertex attribute
    a_position_loc = gl.get_attrib_location(program, "a_position")
    gl.enable_vertex_attrib_array(a_position_loc)
    
    # Draw each slice
    for offset, length in offsets_length:
        gl.vertex_attrib_pointer(
            a_position_loc,
            size=3,
            type=gl.FLOAT,
            normalized=False,
            stride=0,
            offset=offset * 4  # Float32 alignment
        )
        gl.draw_arrays(gl.TRIANGLES, 0, length)
        
    gl.disable_vertex_attrib_array(a_position_loc)
    gl.enable(gl.CULL_FACE)
```

**Parameters:**
- `numberOfSlices`: The base number of slices. Determines the sampling rate along the view direction [S2, unofficial][S3, unofficial].
- `sliceSpacing`: Distance between slices, often derived from volume resolution or desired sampling density.

**Edge Cases and Pitfalls:**
- **Triangle Winding:** When constructing triangles from slice vertices, ensure the index for the third vertex wraps around correctly (e.g., to index 1) if `k + 1` exceeds the array length to ensure closed polygons [S2, unofficial].
- **Texture Artifacts:** Use `GL_CLAMP_TO_BORDER` for the 3D texture to prevent sampling artifacts caused by texture coordinate transformation outside the volume bounds [S3, unofficial].
- **View Alignment:** The proxy geometry (lines or planes) must be strictly view-aligned to ensure samples can be correctly read from the frame buffer if used for post-processing or picking [S4].

## Subtopics

### [VoxelBlur Depth-of-Field Volume Rendering](10657-voxelblur-depth-of-field-volume-rendering.md)
VoxelBlur is an interactive direct volume rendering method that provides a depth-of-field effect by selectively blurring voxels based on their distance from a predefined focus plane. It extends view-aligned slicing by splitting the rendering pipeline into front-to-back and back-to-front passes, each maintaining an intermediate buffer where blur is applied during compositing. Unlike standard back-to-front blending which relies on native GPU blending, VoxelBlur uses custom shader-based blending to enable the blur effect, allowing users to enhance depth perception and separate focus from context.

### Arbitrary Slicing
This variant uses a plane as proxy geometry to sample arbitrary planes for removing depth complexity and exploring volumes [S4]. The strategy involves rotating the view so that the arbitrary plane is parallel to the image plane (view-aligned), rendering the element to the back buffer, and then displaying the result either in a separate viewport or as a textured polygon in object space [S4]. This allows for interactive slicing without regenerating complex proxy geometry meshes, leveraging the flexibility of view-aligned sampling [S4].

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of slices and $V$ is the number of vertices per slice (dependent on bounding box intersection complexity). Geometry generation is required every frame if the camera moves, adding CPU overhead [S2, unofficial].
- **Memory Complexity:** $O(V_{total})$ for vertex storage, where $V_{total}$ is the total number of vertices for all slices. The 3D texture requires $O(D_x \cdot D_y \cdot D_z)$ memory [S3, unofficial].
- **Performance:** Slower than static ray casting for fixed viewpoints due to geometry regeneration, but more memory efficient than storing large ray-casting buffers [S1, unofficial]. Consistent sampling distance is maintained relative to the camera [S1, unofficial].
- **Strengths:** Consistent sampling distance regardless of viewpoint; leverages standard GPU rasterization pipeline; easy to integrate with other proxy geometry techniques (e.g., picking) [S1, unofficial][S4].
- **Limitations:** Geometry must be regenerated on view change; potential for sampling artifacts if slice spacing is too large relative to volume features; requires careful handling of texture clamping [S3, unofficial][S2, unofficial].
- **When to use:** Interactive exploration where consistent sampling density is critical; scenarios requiring integration with other view-aligned proxy geometry operations (e.g., arbitrary slicing, picking) [S4].
- **When not to use:** Static viewpoints where ray casting is more efficient; volumes with extreme anisotropy where slice alignment may cause aliasing if not handled carefully.

## Background
View-Aligned Slicing is a proxy geometry-based volume rendering technique. It was discussed in the context of integrating dynamic deformations into interactive volume visualization, highlighting the flexibility of using proxy geometry for various sampling strategies including ray-casting and arbitrary slicing [S4]. The technique is also implemented in various educational and open-source WebGL volume renderers, demonstrating its practicality for real-time rendering [S2, unofficial][S3, unofficial]. It builds on the principles of slice-based rendering but emphasizes dynamic alignment with the view to maintain sampling consistency [S1, unofficial].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S3] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
[S4] Integrating Dynamic Deformations into Interactive Volume Visualization, https://doi.org/10.2312/vissym.eurovis06.219-226, fetched 2026-10-31
