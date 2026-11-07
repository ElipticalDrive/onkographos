# View-Aligned Slicing


## Overview
View-aligned slicing is a proxy geometry method that divides a 3D volume into parallel slices and rotates the texture coordinates to align these slices with the view plane, ensuring consistent sampling distance regardless of camera orientation [S2, unofficial]. Unlike static slice methods that suffer from intensity variations and changing sampling distances as the camera moves, this approach maintains uniform resolution by keeping the proxy geometry stationary while transforming the texture lookup [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: 3D volumetric scalar data, camera position and orientation, number of slices $n$.
*   **Output**: 2D image composited via hardware alpha blending.

**Data Structures**
*   **3D Texture**: Stores the volume data. Clamping mode must be set to `GL_CLAMP_TO_BORDER` to prevent artifacts during texture coordinate transformation [S2, unofficial].
*   **Proxy Geometry**: A unit quad (or triangle fan) representing a single slice. The geometry itself remains stationary in screen space (typically the X-Y plane).
*   **Shader Uniforms**: Passed per slice in the rendering loop:
    *   `sliceNumber` (float): Index of the current slice.
    *   `totalSlices` (float): Total number of slices $n$.
    *   `MVPMatrix` (Matrix4f): Model-View-Projection matrix.
    *   `ModelMatrix` (Matrix4f): Model matrix.
    *   `ViewMatrix` (Matrix4f): View matrix.
    *   `TextureMatrix` (Matrix4f): Matrix for transforming texture coordinates [S2, unofficial].

**Parameters**
*   `numberOfSlices` ($n$): Determines the sampling rate. Higher values increase accuracy but reduce performance [S2, unofficial].

**Steps**
1.  **Volume Loading**: Load the 3D volume data into a 3D texture [S2, unofficial].
2.  **Geometry Setup**: Scale a unit cube to match the volume's physical dimensions [S2, unofficial].
3.  **Slice Division**: Divide the volume into $n$ slices along the z-direction [S2, unofficial].
4.  **View Alignment**: Align slices to the view plane (X-Y plane). This is achieved by using a rotation matrix that transforms texture coordinates rather than the geometry itself. The slices (unit quads) stay in the same position, and only the texture content changes [S2, unofficial].
5.  **Vertex Processing**: In the vertex shader, transform the texture coordinates using the rotation matrix derived from the view orientation [S2, unofficial].
6.  **Rendering Loop**: Iterate through each slice. Calculate the slice position and pass it to the shader [S2, unofficial].
7.  **Fragment Shading**:
    *   Sample the 3D texture using the transformed texture coordinates to get scalar value $v$ [S2, unofficial].
    *   Map $v$ to color and opacity using a transfer function. The transfer function can be defined by a manual array of points fitted with a cubic spline [S2, unofficial].
8.  **Compositing**: Composite fragments into the frame buffer using hardware alpha blending in back-to-front order [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
volume_data = load_3d_volume("data.raw")
texture_3d = create_3d_texture(volume_data)
texture_3d.set_clamping_mode(GL_CLAMP_TO_BORDER) # Avoid artifacts [S2, unofficial]
n = 64 # numberOfSlices [S2, unofficial]

# Transfer Function Setup
tf_points = [(0.0, 0.0, 0.0), (0.5, 1.0, 0.5), (1.0, 1.0, 1.0)] # Example points
tf_spline = fit_cubic_spline(tf_points) # [S2, unofficial]

# Geometry Setup
# A unit quad is used for each slice. 
# The geometry is stationary in view space (X-Y plane).
# The volume is conceptually divided into n slices in z-direction [S2, unofficial].

for slice_idx in range(n):
    # Calculate slice-specific parameters
    slice_number = float(slice_idx)
    total_slices = float(n)
    
    # Get current camera matrices
    mvp_matrix = get_mvp_matrix()
    model_matrix = get_model_matrix()
    view_matrix = get_view_matrix()
    
    # Calculate Texture Matrix for view alignment
    # This matrix rotates the texture coordinates to align with the view plane
    # keeping the geometry stationary [S2, unofficial]
    texture_matrix = calculate_view_alignment_matrix(view_matrix, slice_idx, n)
    
    # Render the slice proxy geometry (unit quad)
    set_uniform("sliceNumber", slice_number)
    set_uniform("totalSlices", total_slices)
    set_uniform("MVPMatrix", mvp_matrix)
    set_uniform("ModelMatrix", model_matrix)
    set_uniform("ViewMatrix", view_matrix)
    set_uniform("TextureMatrix", texture_matrix)
    
    # Vertex Shader Logic (conceptual)
    # transformed_tex_coord = TextureMatrix * original_tex_coord
    
    # Fragment Shader Logic (conceptual)
    # v = sample_3d_texture(texture_3d, transformed_tex_coord)
    # color, alpha = apply_transfer_function(tf_spline, v)
    
    # Hardware Compositing (Back-to-Front)
    # Blend function: GL_ONE, GL_ONE_MINUS_SRC_ALPHA
    # C_out = C_src + (1 - alpha_src) * C_dst
    # alpha_out = alpha_src + (1 - alpha_src) * alpha_dst
    render_slice()
```

**Edge Cases and Pitfalls**
*   **Texture Artifacts**: If `GL_CLAMP_TO_BORDER` is not used, texture coordinate transformations may cause sampling artifacts at the edges of the volume [S2, unofficial].
*   **Sorting**: Vertices must be sorted in counter-clockwise order to ensure correct rasterization [S1, unofficial].
*   **Precision**: The consistency of sampling distance relies on the accuracy of the texture coordinate transformation [S1, unofficial].

## Subtopics

### 2D Visualization from 3D Grid (Slow)
Renders the volume by projecting 2D slices without view alignment, leading to variable sampling density. Use when performance is critical and view independence is not required. [Link](1507-2d-visualization-from-3d-grid-slow.md)

### Hybrid Ray Tracer
Combines ray tracing for specific features with rasterization. Use for high-quality specular effects or shadows in addition to volume rendering. [Link](2308-hybrid-ray-tracer.md)

### Mixed Geometry Rendering
Uses a mix of proxy geometry and ray marching. Use when the volume has both dense and sparse regions. [Link](3442-mixed-geometry-rendering.md)

### Texture-Based Volume Rendering
Uses ray casting through 3D textures directly. Use for simple implementations without proxy geometry overhead. [Link](4707-texture-based-volume-rendering.md)

### Image-based rendering integration
Integrates volume rendering with image-based lighting or post-processing. Use for realistic lighting effects. [Link](6969-image-based-rendering-integration.md)

### Vector Icon Volume Rendering
Renders vector fields using icons. Use for flow visualization. [Link](7693-vector-icon-volume-rendering.md)

## Operating characteristics
*   **Time Complexity**: $O(n \cdot R)$, where $n$ is the number of slices and $R$ is the resolution of the proxy geometry (typically constant for a unit quad). The cost is dominated by the number of slices and fragment shader operations.
*   **Memory Complexity**: $O(V)$ for the 3D texture, where $V$ is the volume of the data.
*   **Performance**: Slower than static slice methods due to the per-slice transformation and sorting, but more memory efficient than full ray casting [S1, unofficial].
*   **Strengths**: Consistent sampling distance regardless of camera angle; minimizes sampling artifacts; leverages hardware alpha blending [S1, unofficial][S2, unofficial].
*   **Limitations**: Requires sorting of slices; texture coordinate transformation can be complex to implement correctly; not suitable for highly anisotropic volumes where slice alignment is critical [S1, unofficial][S2, unofficial].
*   **When to use**: When view-independent sampling quality is required and the volume is relatively isotropic.
*   **When not to use**: When extreme performance is needed and view-dependent artifacts are acceptable; or for volumes with strong directional features that do not align well with planar slicing.

## Background
View-aligned slicing is a technique developed to address the inconsistencies of static slice-based rendering. It was implemented in the open-source project `SliceBasedVolumeRendering` by Abhishek Trip [S2, unofficial]. The method builds on the principles of proxy geometry rendering, where the volume is approximated by a set of polygons [S1, unofficial].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
