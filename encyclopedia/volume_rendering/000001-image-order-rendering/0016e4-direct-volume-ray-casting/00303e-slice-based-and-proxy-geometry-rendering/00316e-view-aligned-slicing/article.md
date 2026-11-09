# View-Aligned Slicing

Also called: View-Aligned Proxy Geometry Sampling, View-Aligned Slice Rendering

## Overview
View-Aligned Slicing is a direct volume rendering technique that decomposes a 3D volume into a series of 2D triangular proxy polygons aligned with the view direction [S2, unofficial][S3, unofficial]. It solves the problem of inconsistent sampling density and artifacts inherent in fixed-grid slicing by dynamically regenerating geometry to maintain consistent slice thickness relative to the camera [S1, unofficial]. Unlike standard ray casting which samples volume textures directly, this method rasterizes explicit geometry, requiring geometry regeneration on every view change [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volumetric data (scalar field), camera parameters (view and projection matrices), number of slices ($N$).
*   **Output:** 2D image composited into the frame buffer.

**Data Structures:**
*   **Volume Texture:** A 3D texture (`sampler3D`) storing the scalar data. Clamping mode must be `GL_CLAMP_TO_BORDER` to prevent artifacts during coordinate transformation [S3, unofficial].
*   **Transfer Function:** A 1D texture (`sampler2D`) mapping scalar values to color ($C$) and opacity ($\alpha$) [S1, unofficial][S3, unofficial].
*   **Vertex Buffer (`a_position`):** A `Float32Array` containing flat vertex coordinates $[x, y, z]$ for all slices concatenated together [S2, unofficial].
*   **Offset/Length Array (`offsets_length`):** An array of tuples $[offset, length]$, where $offset$ is the byte offset in the vertex buffer and $length$ is the number of vertices for each slice [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Load 3D volume data into a 3D texture [S3, unofficial].
    *   Load transfer function into a 1D texture [S3, unofficial].
    *   Calculate the total number of slices to generate. Sources indicate generating $2N$ slices during initialization and $N$ slices during the draw pass [S2, unofficial]. Let $N$ be the base number of slices [S2, unofficial][S3, unofficial].

2.  **Geometry Generation (Per Frame):**
    *   Compute the view-aligned geometry by intersecting slicing planes with the volume's bounding box [S1, unofficial].
    *   The volume is conceptually divided into $N$ slices along the z-direction [S3, unofficial].
    *   A rotation matrix is applied to align these slices with the view plane (typically the X-Y plane) [S3, unofficial]. This can be achieved by transforming the texture coordinates in the vertex shader while keeping the proxy geometry (unit quad/cube) stationary, or by rotating the geometry itself [S3, unofficial].
    *   Vertices are sorted in counter-clockwise order [S1, unofficial].
    *   Construct polygon primitives from the centroid as a triangle fan [S1, unofficial].
    *   *Pitfall:* When constructing triangles from slice vertices, if the index $k+1$ exceeds the array length, it wraps around to index 1 to ensure closed polygons [S2, unofficial].
    *   Update the vertex attribute buffer `a_position` with the newly generated slice data [S1, unofficial].

3.  **Rendering Loop:**
    *   Enable blending: `gl.enable(gl.BLEND)` [S2, unofficial].
    *   Set blend function for the Over operator: Source = `GL_ONE`, Destination = `GL_ONE_MINUS_SRC_ALPHA` [S1, unofficial][S2, unofficial].
    *   Disable face culling during rendering to handle arbitrary orientations: `gl.disable(gl.CULL_FACE)` [S2, unofficial].
    *   Bind the vertex buffer and enable the vertex attribute array for `a_position` [S2, unofficial].
    *   Iterate through the `offsets_length` array. For each slice $i$:
        *   Set the vertex pointer: size 3, type `gl.FLOAT`, normalized false, stride 0, offset $= offset_i \times 4$ bytes (for Float32 alignment) [S2, unofficial].
        *   Draw the slice using `gl.drawArrays(gl.TRIANGLES, 0, length_i)` [S2, unofficial].
    *   Re-enable face culling if required for subsequent passes: `gl.enable(gl.CULL_FACE)` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_view_aligned_slices(volume_texture, transfer_fn, camera_transform, num_slices):
    """
    Renders a 3D volume using view-aligned slicing proxy geometry.
    
    Args:
        volume_texture: 3D texture object containing scalar data.
        transfer_fn: 1D texture object for color/opacity mapping.
        camera_transform: Object containing lookAtMatrix, perspectiveMatrix, translationMatrix.
        num_slices: Base number of slices N.
    """
    # 1. Setup OpenGL State
    gl.enable(gl.BLEND)
    gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)
    gl.disable(gl.CULL_FACE) # Disable culling for arbitrary view alignment
    
    # 2. Bind Shaders and Textures
    use_program(volume_shader_program)
    bind_texture_3d(volume_texture, unit=0)
    bind_texture_1d(transfer_fn, unit=1)
    
    # 3. Generate Geometry
    # Note: Some implementations generate 2*N during init, N during draw [S2, unofficial]
    # Here we generate N slices for the current frame
    vertex_data, offsets_length = generate_view_aligned_slices(
        slice_count=num_slices,
        look_at_matrix=camera_transform.lookAtMatrix,
        translation_matrix=camera_transform.translationMatrix
    )
    
    # 4. Update Vertex Buffer
    update_attribute_buffer("a_position", vertex_data)
    
    # 5. Render Loop
    a_position_loc = get_attribute_location("a_position")
    gl.enableVertexAttribArray(a_position_loc)
    bind_buffer(gl.ARRAY_BUFFER, get_buffer_id("a_position"))
    
    for offset, length in offsets_length:
        # Set vertex pointer: 3 floats, no normalization, stride 0, byte offset
        vertex_pointer(
            location=a_position_loc,
            size=3,
            type=gl.FLOAT,
            normalized=False,
            stride=0,
            offset=offset * 4 # Float32 is 4 bytes
        )
        
        # Draw triangles for this slice
        draw_arrays(primitive_type=gl.TRIANGLES, first=0, count=length)
        
    gl.disableVertexAttribArray(a_position_loc)
    gl.enable(gl.CULL_FACE) # Restore culling state
```

**Formulas and Parameters:**
*   **Sampling Rate:** Determined by `numberOfSlices` ($N$) [S3, unofficial].
*   **Vertex Offset:** The byte offset for the $i$-th slice in the vertex buffer is calculated as $offset_i \times 4$ bytes, assuming `Float32` vertices [S2, unofficial].
*   **Texture Coordinate Transformation:** The rotation matrix transforms texture coordinates in the vertex shader to align slices with the view plane [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Texture Artifacts:** Using `GL_CLAMP_TO_BORDER` for the 3D texture is critical to avoid artifacts caused by the transformation of texture coordinates [S3, unofficial].
*   **Geometry Closure:** Triangle construction must handle index wrapping to ensure closed polygons [S2, unofficial].
*   **View Alignment:** The proxy geometry must be strictly view-aligned to ensure samples can be correctly read from the frame buffer in subsequent interaction steps (e.g., picking) [S4].

## Subtopics

### Arbitrary Slicing
Arbitrary Slicing extends View-Aligned Slicing by using planes as proxy geometry to sample arbitrary planes within the volume, rather than just view-aligned slices for full rendering [S4]. It removes depth complexity when exploring volumes [S4]. The plane is view-aligned (rotated to be parallel to the image plane) and rendered to the back buffer, allowing results to be displayed in a separate viewport or as a textured polygon in object space [S4]. This technique enables interactive probing and slicing without deforming the mesh of the proxy geometry [S4].

## Operating characteristics
*   **Time Complexity:** Geometry generation is $O(N)$ where $N$ is the number of slices. Rasterization is $O(P)$ where $P$ is the number of pixels covered by the proxy geometry.
*   **Memory Complexity:** $O(V)$ for the volume texture and $O(N \cdot V_{poly})$ for the vertex buffer, where $V_{poly}$ is the vertex count per slice.
*   **Performance:** Slower than fixed-grid methods due to per-frame geometry regeneration [S1, unofficial]. However, it is more memory efficient than full ray casting for large volumes [S1, unofficial].
*   **Strengths:** Consistent sampling distance regardless of viewpoint [S1, unofficial]. Reduces sampling artifacts compared to fixed-grid slicing [S3, unofficial].
*   **Limitations:** Requires geometry regeneration on every view change [S2, unofficial]. Not suitable for dynamic volume data without frequent buffer updates.
*   **When to use:** Interactive visualization where consistent sampling quality is prioritized over raw frame rate, or when exploring volumes with arbitrary slicing planes [S4].
*   **When not to use:** Real-time applications with strict performance constraints where fixed-grid slicing is sufficient, or when the volume data changes every frame.

## Background
View-Aligned Slicing is part of the broader family of proxy geometry rendering methods for volume visualization. It was developed to address the limitations of fixed-grid slicing, such as inconsistent sampling density and artifacts when the camera moves [S1, unofficial][S3, unofficial]. The technique leverages hardware rasterization capabilities to perform volume sampling, integrating with standard graphics pipelines [S2, unofficial][S4]. It is often discussed in the context of interactive volume visualization and dynamic deformations [S4].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S3] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
[S4] Integrating Dynamic Deformations into Interactive Volume Visualization, https://doi.org/10.2312/vissym.eurovis06.219-226, fetched 2026-10-31
