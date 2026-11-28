# View-Aligned Slicing

Also called: 3D-Texture Based Slicing, View-Aligned 3D Texture Slicing, View-Aligned Proxy Geometry Sampling, View-Aligned Slice Rendering

## Overview
View-aligned slicing is a direct volume rendering technique that decomposes a 3D volume into a series of 2D planar slices (proxy geometry) aligned with the camera's view direction [S2, S3, S4, unofficial]. Unlike standard ray casting which samples the volume texture directly per pixel, this method generates explicit geometric primitives (triangles or quads) that are rasterized by the GPU [S2, unofficial][S5]. The primary advantage is consistent sampling distance relative to the camera, which minimizes sampling artifacts and maintains uniform slice thickness regardless of viewing angle [S4, S1, unofficial]. It builds on [Proxy Geometry and Slicing](15424-proxy-geometry-and-slicing.md) by dynamically regenerating geometry every frame to maintain view alignment [S2, S3, unofficial].

## How it works
The method renders the volume by intersecting parallel slicing planes with the volume bounding box, generating billboards that face the camera [S3, unofficial]. These billboards are rasterized, and fragment shaders sample the 3D volume texture at interpolated 3D texture coordinates [S2, unofficial].

### Inputs and Outputs
*   **Inputs**: 3D volume data (scalar field), camera position (`eyePos`), view matrix (`mtx`), volume dimensions (`res`), and sampling rate (`sampleRate` or `numberOfSlices`) [S3, unofficial].
*   **Outputs**: 2D image with color and opacity composited via alpha blending [S3, unofficial].

### Data Structures
1.  **Volume Texture (`tex3D`)**:
    *   Format: `GL_RGBA32F` (or 8-bit precision in some implementations [S7, unofficial]).
    *   Dimensions: `rawinf->res.x`, `rawinf->res.y`, `rawinf->res.z`.
    *   Wrap Mode: `GL_CLAMP_TO_BORDER` to avoid artifacts from texture coordinate transformation [S4, S3, unofficial].
    *   Filter: `GL_NEAREST` [S3, unofficial].
    *   Binding: `GL_TEXTURE0`.
2.  **Transfer Function (`tex1D`)**:
    *   Format: `GL_RGBA32F`.
    *   Size: 256 entries [S3, S7, unofficial].
    *   Wrap Mode: `GL_CLAMP`.
    *   Filter: `GL_LINEAR`.
    *   Binding: `GL_TEXTURE1`.
3.  **Vertex Buffer (`a_position`)**:
    *   Type: `Float32Array` (flat array of `[x, y, z]` coordinates) [S2, unofficial].
    *   Layout: Concatenated vertices for all slices.
4.  **Offset Array (`offsets_length`)**:
    *   Type: Array of tuples `[offset, length]` [S2, unofficial].
    *   `offset`: Byte offset in the vertex buffer (multiplied by 4 for `Float32` alignment).
    *   `length`: Number of vertices for that specific slice.

### Algorithm Steps

1.  **Initialization**:
    *   Load 3D volume data into `tex3D` [S4, S3, unofficial].
    *   Load transfer function into `tex1D` [S3, unofficial].
    *   Define volume bounds: $x_{bound} = res.x$, $y_{bound} = res.y$, $z_{bound} = res.z$ [S3, unofficial].
    *   Define the 8 corners of the volume bounding box in local coordinates:
        $$
        corner[8][3] = \left\{ \pm \frac{x_{bound}}{2}, \pm \frac{y_{bound}}{2}, \pm \frac{z_{bound}}{2} \right\}
        $$
        [S3, unofficial].

2.  **Geometry Generation (Per Frame)**:
    *   Calculate the distance $d$ from the eye to the origin along the view normal (z-axis of the view matrix `mtx`):
        $$
        d = mtx[2] \cdot eyePos[0] + mtx[6] \cdot eyePos[1] + mtx[10] \cdot eyePos[2]
        $$
        [S3, unofficial].
    *   Find the `furthest` and `nearest` intersection distances by iterating over the 8 corners:
        $$
        distance_i = \left| -mtx[2] \cdot corner[i][0] - mtx[6] \cdot corner[i][1] - mtx[10] \cdot corner[i][2] + d \right|
        $$
        [S3, unofficial].
        *   $nearest = \min(distance_i)$
        *   $furthest = \max(distance_i)$
    *   Calculate the step size (slice thickness):
        $$
        \delta D = \frac{furthest - nearest}{sampleRate}
        $$
        [S3, unofficial].
    *   Calculate the billboard size (diagonal of the bounding box):
        $$
        size = 2 \cdot \sqrt{x_{bound}^2 + y_{bound}^2 + z_{bound}^2}
        $$
        [S3, unofficial].
    *   Generate slice geometry:
        *   Loop from $i = furthest$ down to $nearest$ with step $\delta D$ (back-to-front order) [S3, unofficial].
        *   Calculate billboard center:
            $$
            billboardCenter[0] = eyePos[0] - i \cdot mtx[2]
            $$
            $$
            billboardCenter[1] = eyePos[1] - i \cdot mtx[6]
            $$
            $$
            billboardCenter[2] = eyePos[2] - i \cdot mtx[10]
            $$
            [S3, unofficial].
        *   Construct a polygon primitive (triangle fan or quad) centered at `billboardCenter` with `size` [S3, S1, unofficial].
        *   Assign 3D texture coordinates to vertices. The coordinates map the unit quad to the volume bounds [S4, unofficial]. In some implementations, a rotation matrix transforms texture coordinates in the vertex shader to align with the view, keeping the slice geometry stationary while changing the texture content [S4, unofficial].
        *   Store vertices in `a_position` and record `[offset, length]` in `offsets_length` [S2, unofficial].

3.  **Rendering**:
    *   Enable blending and set blend function to standard alpha compositing [S3, unofficial].
    *   Bind vertex buffer and enable vertex attribute array for `a_position` [S2, unofficial].
    *   Iterate through `offsets_length`:
        *   Set vertex pointer with size 3, type `FLOAT`, normalized false, stride 0, and offset calculated as `offset * 4` (for `Float32` byte alignment) [S2, unofficial].
        *   Draw slice using triangle primitives with the recorded vertex count [S2, unofficial].
    *   Fragment Shader:
        *   Sample `tex3D` at interpolated 3D texture coordinates to get scalar value [G].
        *   Map scalar value to color and opacity using a 1D transfer function texture [S4, unofficial].
        *   Composite using alpha blending [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_view_aligned_slices(number_of_slices, eye_pos, mtx, res):
    # 1. Define volume bounds and 8 corners
    x_bound, y_bound, z_bound = res.x, res.y, res.z
    corners = []
    for dx in [-1, 1]:
        for dy in [-1, 1]:
            for dz in [-1, 1]:
                corners.append([dx * x_bound / 2, dy * y_bound / 2, dz * z_bound / 2])

    # 2. Calculate distance from eye to origin along view normal
    d = mtx[2] * eye_pos[0] + mtx[6] * eye_pos[1] + mtx[10] * eye_pos[2]

    # 3. Find nearest and furthest intersection distances
    distances = []
    for corner in corners:
        dist = abs(-mtx[2] * corner[0] - mtx[6] * corner[1] - mtx[10] * corner[2] + d)
        distances.append(dist)
    nearest = min(distances)
    furthest = max(distances)

    # 4. Calculate step size and billboard size
    delta_d = (furthest - nearest) / number_of_slices
    size = 2.0 * sqrt(x_bound**2 + y_bound**2 + z_bound**2)

    # 5. Generate slice geometry
    vertices = []
    offsets = []
    for i in range(furthest, nearest - delta_d, -delta_d):
        # Billboard center calculation using view direction components
        center = [
            eye_pos[0] - i * mtx[2],
            eye_pos[1] - i * mtx[6],
            eye_pos[2] - i * mtx[10]
        ]
        # Construct polygon primitive (triangle fan or quad) centered at billboardCenter with size
        # Assign 3D texture coordinates mapping unit quad to volume bounds
        # Transform texture coordinates in vertex shader to align with view [S4, unofficial]
        slice_vertices = construct_billboard(center, size)
        vertices.extend(slice_vertices)
        offsets.append([len(vertices) * 4, len(slice_vertices)]) # offset in bytes for Float32

    return vertices, offsets

def render_slices(vertices, offsets, tex3d, tex1d):
    # Enable blending and set blend function to standard alpha compositing [S3, unofficial]
    enable_blending()
    set_blend_func(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)
    
    # Bind vertex buffer and enable vertex attribute array for a_position [S2, unofficial]
    bind_buffer(vertices)
    enable_vertex_attribute("a_position")
    
    # Iterate through offsets_length to draw each slice individually [S2, unofficial]
    for offset, length in offsets:
        # Set vertex pointer: size=3, type=FLOAT, normalize=False, stride=0, offset=offset*4 [S2, unofficial]
        set_vertex_pointer(location=0, size=3, type=FLOAT, normalize=False, stride=0, offset=offset * 4)
        # Draw slice using triangle primitives [S2, unofficial]
        draw_arrays(TRIANGLES, start=0, count=length)
```

### Parameters
*   `numberOfSlices` / `sampleRate`: Determines the number of slices [S4, unofficial].
*   `nearClip`: Near plane of the orthographic projection, default 200.0 [S3, unofficial].
*   `farClip`: Far plane of the orthographic projection, default 600.0 [S3, unofficial].
*   `viewBound`: Half-width of the orthographic projection box, default 50, adjustable by keys 'z' (multiply by 0.9) and 'x' (multiply by 10/9) [S3, unofficial].
*   `boundary`: Integer flag passed to shader, default 0, toggled by keys '1' (0) and '2' (1) [S3, unofficial].

### Edge Cases and Pitfalls
*   **Texture Artifacts**: Using `GL_CLAMP_TO_BORDER` is critical when transforming texture coordinates to avoid sampling artifacts at slice edges [S4, S3, unofficial].
*   **Index Wrapping**: When constructing triangles from slice vertices, ensure the index for the third vertex wraps correctly (e.g., to index 1) if `k + 1` exceeds the array length to form closed polygons [S2, unofficial].
*   **View Alignment**: The proxy geometry must be strictly view-aligned to ensure samples can be correctly read from the frame buffer if post-processing or compositing relies on buffer reads [S5].
*   **Performance Management**: The number of slices is automatically reduced during rotation to guarantee interactive rates [S7, unofficial].

## Subtopics

### [Object-Aligned Slicing](6531-object-aligned-slicing.md)
Object-aligned slicing divides the volume into slices along the principal axes (x, y, z) rather than the view direction. It uses fixed texture coordinates assigned to slice vertices and avoids per-ray intersection calculations. It is distinguished from its parent [View-Aligned Slicing](6530-view-aligned-slicing.md) by its simplicity and speed, but suffers from higher memory consumption and visual artifacts due to inconsistent sampling distances and stack swapping.

### Arbitrary Slicing
Arbitrary slicing uses a plane as proxy geometry to sample arbitrary planes for removing depth complexity and exploring volumes [S5]. The plane is view-aligned (rotated to be parallel to the image plane) and rendered to the back buffer [S5]. Results can be displayed in a separate viewport or as a textured polygon in object space [S5]. This allows for interactive probing and slicing without regenerating the full volume geometry [S5].

### Ray-to-Gravity-Center (R-GC) Volume Slicing
R-GC is a volume slicing strategy where parallel cutting planes are oriented perpendicular to the straight line connecting the point of reflection to the gravity center of the volume domain [S6]. It is used to generate Layered Polygonal Area Lights (LPALs) for indirect illumination [S6]. Compared to Ray-of-Regular-Reflection (R-RR) slicing, R-GC ensures LPALs face properly to the point of reflection regardless of the regular reflection direction, correctly rendering the upper part of the reflected appearance when the reflection point is apart from the volume [S6].

### GPU Volume Rendering Implementation
This variant implements view-aligned slicing on the GPU with pre-integration [S7, unofficial]. It uses two levels of pre-integration: simplified (neglecting color attenuation, faster) for interactive transfer function editing, and full pre-integration for final rendering [S7, unofficial]. Volume shading can be enabled using a per-fragment Phong lighting model based on the gradient of the scalar field [S7, unofficial]. The number of slices is automatically reduced during rotation to guarantee interactive rates [S7, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: Interactive rates are achievable; the number of slices is automatically reduced during rotation to guarantee interactive rates [S7, unofficial].
*   **Strengths**: Consistent sampling distance, minimal aliasing artifacts compared to object-aligned methods, efficient use of GPU rasterization [S1, S4, unofficial].
*   **Limitations**: Requires geometry regeneration every frame [S2, unofficial].
*   **When to use**: Interactive volume visualization where view alignment is critical for quality and performance is acceptable [S7, unofficial].
*   **When not to use**: When extreme lighting complexity (e.g., accurate indirect illumination) is required without additional techniques, or when memory bandwidth is severely constrained [S6][S1, unofficial].

## Background
View-aligned slicing was implemented in systems like "Scientific-visualization-Slicing-volume-rendering" [S3, unofficial] and "SliceBasedVolumeRendering" [S4, unofficial]. The technique is discussed in the context of proxy geometry for interactive volume visualization [S5] and in implementations of pre-integrated volume rendering [S7, unofficial].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S3] extrovert7986/Scientific-visualization-Slicing-volume-rendering, https://github.com/extrovert7986/Scientific-visualization-Slicing-volume-rendering, fetched 2026-10-26 (unofficial)
[S4] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
[S5] Integrating Dynamic Deformations into Interactive Volume Visualization, https://doi.org/10.2312/vissym.eurovis06.219-226, fetched 2026-10-31
[S6] Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights, https://doi.org/10.1111/cgf.13851, fetched 2026-10-26
[S7] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
