# View-Aligned Slicing

Also called: View-Aligned Proxy Geometry Sampling, View-Aligned Slice Rendering

## Overview
View-Aligned Slicing is a direct volume rendering technique that decomposes a 3D volume into a set of 2D triangular proxy geometry slices aligned with the view direction [S2, unofficial][S4]. It solves the problem of maintaining consistent sampling density and minimizing sampling artifacts inherent in standard ray casting by dynamically regenerating geometry per frame based on the camera transform [S3, unofficial][S4]. Unlike standard ray casting which samples the volume texture directly via shader loops, this method rasterizes explicit geometry, requiring regeneration of geometry when the view changes [S2, unofficial].

## How it works
The method operates on the fundamental loop structure of [Image-order rendering](16-image-order-rendering.md): for each pixel, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, intersected with the volume domain, and sampled/composited along its path [G]. The traversal order is determined by the pixel's position in the image plane [G].

The core computational engine consists of four sequential stages applied to each ray [S5, S10, S37, unofficial]:

1.  **Ray Generation and Discretization**: The ray segment $[t_{start}, t_{end}]$ intersecting the volume is determined, typically by intersecting the ray with the volume's bounding box [S1, unofficial][S8, S16, unofficial]. The segment is discretized into $N$ samples at equidistant intervals $\Delta t = (t_{end} - t_{start}) / N$ [S5, S10, S17, unofficial].
2.  **Sampling and Interpolation**: At each sample point $\mathbf{p}_i = r(t_i)$, the scalar value $V(\mathbf{p}_i)$ is retrieved from the volume data. This is typically performed by trilinear interpolation of the eight surrounding voxels [S9, S11, S47, unofficial].
3.  **Classification (Transfer Function)**: The scalar value $V(\mathbf{p}_i)$ is mapped to optical properties: color $C_i$ and opacity $\alpha_i$ [S2, S2789]. The opacity is often corrected for the step size $\Delta t$ to ensure physical consistency, using the Beer-Lambert law approximation:
    $$ \alpha_i = 1 - e^{-\sigma_i \Delta t} $$
    where $\sigma_i$ is the extinction coefficient derived from the scalar value [S24, S30, unofficial].
4.  **Compositing**: The samples are accumulated from front to back (closest to the eye to furthest) using the front-to-back alpha compositing equation [S1, unofficial][S2, S13, unofficial][S61]:
    $$ C_{out} = C_{in} + (1 - \alpha_{in}) \cdot C_{sample} \cdot \alpha_{sample} $$
    $$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample} $$
    where $C_{in}$ and $\alpha_{in}$ are the accumulated color and opacity from previous samples, and $C_{sample}$ and $\alpha_{sample}$ are the color and opacity of the current sample [S52, S60, unofficial]. The process initializes with $C_{in} = 0$ and $\alpha_{in} = 0$ [S15, S46, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   3D Volume Data: Stored in a 3D texture [S3, unofficial].
    *   Transfer Function: Stored in a 1D or 2D texture [S2, S3, unofficial].
    *   Camera Parameters: `lookAtMatrix`, `perspectiveMatrix`, `translationMatrix` [S2, S3, unofficial].
    *   Rendering Parameters: `numberOfSlices` (base count) [S2, S3, unofficial].
*   **Outputs**:
    *   Final Image: Rendered to the frame buffer [S2, unofficial][S4].

### Data Structures
*   **Vertex Buffer (`a_position`)**: A `Float32Array` containing flat vertex coordinates $[x, y, z]$ for all slices concatenated together [S2, unofficial].
*   **Offsets Array (`offsets_length`)**: An array of tuples `[offset, length]` where `offset` is the byte offset in the vertex buffer and `length` is the number of vertices in that slice [S2, unofficial].
*   **Shader Uniforms**:
    *   `u_matrix_projection`: Perspective matrix [S2, unofficial].
    *   `u_matrix_camera_translation`: Look-at matrix [S2, unofficial].
    *   `u_matrix_translation`: Translation matrix [S2, unofficial].
    *   `u_volume_size`: Volume dimensions [S2, unofficial].
    *   `sliceNumber`: Current slice index (float) [S3, unofficial].
    *   `totalSlices`: Total number of slices (float) [S3, unofficial].
    *   `MVPMatrix`, `ModelMatrix`, `ViewMatrix`, `TextureMatrix`: Per-slice matrices [S3, unofficial].

### Algorithm Steps

1.  **Initialization**:
    *   Load 3D volume data into a 3D texture [S3, unofficial].
    *   Set 3D texture clamping mode to `GL_CLAMP_TO_BORDER` to avoid artifacts from texture coordinate transformation [S3, unofficial].
    *   Generate initial proxy geometry with `numberOfSlices * 2` slices [S2, unofficial].
    *   Create vertex attribute buffer `a_position` with `gl.DYNAMIC_DRAW` usage hint [S2, unofficial].

2.  **Geometry Generation (Per Frame)**:
    *   Calculate the number of slices to generate as `this.numberOfSlices` during the draw pass [S2, unofficial].
    *   Compute view-aligned geometry by calling `generateViewAlignedSlices` with the slice count, `lookAtMatrix`, and `translationMatrix` to obtain vertex data and offset/length pairs [S2, unofficial].
    *   The geometry generation involves:
        *   Dividing the volume into $n$ slices in the z-direction, where $n$ is the sampling rate [S3, unofficial].
        *   Intersecting slicing planes with the volume bounding box [S1, unofficial].
        *   Sorting the resulting vertices in counter-clockwise order [S1, unofficial].
        *   Constructing a polygon primitive from the centroid as a triangle fan [S1, unofficial].
        *   *Pitfall*: When constructing triangles from slice vertices, the index for the third vertex wraps around to index 1 if `k + 1` exceeds the array length to ensure closed polygons [S2, unofficial].

3.  **Rendering Loop**:
    *   Update the vertex attribute buffer `a_position` with the newly generated slice data using `updateAttribute` [S2, unofficial].
    *   Bind the vertex buffer and enable the vertex attribute array for `a_position` [S2, unofficial].
    *   Iterate through the `offsets` array:
        *   Set the vertex pointer for each slice with size 3, type `gl.FLOAT`, normalized false, stride 0, and offset calculated as `offset * 4` (for Float32 byte alignment) [S2, unofficial].
        *   Draw each slice individually using `gl.drawArrays` with `gl.TRIANGLES` [S2, unofficial].
    *   *Note*: The slices are view-aligned (in the X-Y plane) and stack up in the negative z direction [S3, unofficial]. To create view-aligned slices, a rotation matrix transforms the texture coordinates in the vertex shader, keeping the slice geometry (unit quad) stationary while changing the texture content [S3, unofficial].

4.  **Shading and Compositing**:
    *   Enable blending with `gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)` [S2, unofficial].
    *   Disable culling during rendering to ensure correct front-to-back compositing [S2, unofficial].
    *   In the fragment shader:
        *   Transform texture coordinates [S3, unofficial].
        *   Read the scalar value from the 3D texture [S3, unofficial].
        *   Apply the transfer function (e.g., cubic spline fit on manual points) to determine color and opacity [S3, unofficial].
        *   Output color and alpha for hardware alpha blending [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_view_aligned_slices(number_of_slices, look_at_matrix, translation_matrix):
    """
    Generates vertex data and offsets for view-aligned slices.
    Returns a tuple: (vertices: Float32Array, offsets: List[Tuple[int, int]])
    """
    vertices = []
    offsets = []
    current_offset = 0
    
    # Divide volume into n slices in z-direction
    for i in range(number_of_slices):
        # Intersect slicing plane with volume bounding box
        # Calculate plane position based on slice index and volume dimensions
        plane_position = calculate_plane_position(i, number_of_slices)
        
        # Intersect plane with bounding box to get polygon vertices
        # Sort vertices in counter-clockwise order
        polygon_vertices = intersect_plane_with_bbox(plane_position, look_at_matrix, translation_matrix)
        polygon_vertices = sort_counter_clockwise(polygon_vertices)
        
        # Construct polygon primitive from centroid as triangle fan
        centroid = compute_centroid(polygon_vertices)
        triangles = []
        for k in range(len(polygon_vertices)):
            v0 = polygon_vertices[k]
            v1 = polygon_vertices[(k + 1) % len(polygon_vertices)] # Wrap around to index 1 if k+1 exceeds length
            v2 = centroid
            triangles.extend([v0, v1, v2])
        
        # Append to global vertices list
        vertices.extend(triangles)
        
        # Record offset and length
        length = len(triangles)
        offsets.append((current_offset, length))
        current_offset += length * 4 # 4 bytes per float
        
    return vertices, offsets

def render_view_aligned_slices(volume_texture, transfer_fn_texture, look_at_matrix, perspective_matrix, translation_matrix, number_of_slices):
    """
    Renders the volume using view-aligned slicing.
    """
    # Initialize blending
    enable_blend()
    set_blend_func(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)
    disable_cull_face()
    
    # Bind textures
    bind_texture_3d(volume_texture, unit=0)
    set_texture_clamping(GL_CLAMP_TO_BORDER) # Avoid artifacts from transformation
    bind_texture_1d(transfer_fn_texture, unit=1)
    
    # Generate geometry
    vertices, offsets = generate_view_aligned_slices(number_of_slices, look_at_matrix, translation_matrix)
    
    # Update vertex buffer
    update_attribute_buffer("a_position", vertices)
    enable_vertex_array("a_position")
    bind_buffer(ARRAY_BUFFER, get_buffer_id("a_position"))
    
    # Render loop
    for offset, length in offsets:
        # Set vertex pointer
        # size=3, type=FLOAT, normalize=false, stride=0, offset=offset*4
        vertex_attrib_pointer("a_position", 3, FLOAT, False, 0, offset * 4)
        
        # Draw slice
        draw_arrays(TRIANGLES, 0, length)
        
    enable_cull_face()
```

## Subtopics

### [VoxelBlur Depth-of-Field Volume Rendering](10657-voxelblur-depth-of-field-volume-rendering.md)
VoxelBlur is an interactive direct volume rendering method that provides a depth-of-field effect by selectively blurring voxels based on their distance from a predefined focus plane, simulating real-world camera lens systems. It extends the view-aligned slicing technique by splitting the rendering pipeline into front-to-back and back-to-front passes, each maintaining an intermediate buffer where blur is applied during compositing. Unlike standard back-to-front blending which relies on native GPU blending, VoxelBlur uses custom shader-based blending to enable the blur effect, allowing users to enhance depth perception and separate focus from context.

### Arbitrary Slicing
Using a plane as proxy geometry to sample arbitrary planes for removing depth complexity and exploring volumes [S4]. View-align the slicing plane and render it to the back buffer [S4]. Display the result either in a separate viewport or as a textured polygon in object space [S4].

## Operating characteristics
*   **Time Complexity**: Geometry generation is $O(N \cdot V)$ where $N$ is the number of slices and $V$ is the number of vertices per slice (typically constant for a fixed volume bounding box intersection). Rendering is $O(N)$ draw calls.
*   **Memory Complexity**: $O(N \cdot V)$ for vertex storage.
*   **Measured Performance**: Slower than standard ray casting due to geometry regeneration overhead, but more memory efficient [S1, unofficial].
*   **Strengths**: Consistent sampling distance regardless of viewpoint [S1, unofficial]. Removes depth complexity for arbitrary slicing [S4].
*   **Limitations**: Requires regeneration of geometry when the view changes [S2, unofficial]. Slower than direct ray casting [S1, unofficial].
*   **When to use**: When consistent sampling density is critical, or when arbitrary slicing planes are needed for exploration [S4].
*   **When not to use**: When high performance is required and view changes are frequent [S1, unofficial].

## Background
Originated as a proxy geometry-based volume rendering technique [S2, unofficial][S4]. Developed to address the limitations of standard ray casting in maintaining consistent sampling and handling arbitrary slicing interactions [S4]. Used in interactive volume visualization systems [S4].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S3] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
[S4] Integrating Dynamic Deformations into Interactive Volume Visualization, https://doi.org/10.2312/vissym.eurovis06.219-226, fetched 2026-10-31
