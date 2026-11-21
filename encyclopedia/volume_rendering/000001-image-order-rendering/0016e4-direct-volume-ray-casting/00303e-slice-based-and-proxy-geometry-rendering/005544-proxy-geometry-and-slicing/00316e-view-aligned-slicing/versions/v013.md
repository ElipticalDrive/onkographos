# View-Aligned Slicing

Also called: 3D-Texture Based Slicing, View-Aligned 3D Texture Slicing, View-Aligned Proxy Geometry Sampling, View-Aligned Slice Rendering

## Overview
View-aligned slicing is a direct volume rendering technique that decomposes a 3D volume into a series of 2D planar slices (proxy geometry) aligned with the camera's view direction [S2, S3, S4, unofficial]. Unlike standard ray casting which samples the volume texture directly per pixel, this method generates explicit geometric primitives (triangles or quads) that are rasterized by the GPU [S2, unofficial][S5]. The primary advantage is consistent sampling distance relative to the camera, which minimizes sampling artifacts and maintains uniform slice thickness regardless of viewing angle [S4, S1, unofficial]. It builds on [Proxy Geometry and Slicing](15424-proxy-geometry-and-slicing.md) by dynamically regenerating geometry every frame to maintain view alignment.

## How it works
The method renders the volume by intersecting parallel slicing planes with the volume bounding box, generating billboards that face the camera. These billboards are rasterized, and fragment shaders sample the 3D volume texture at interpolated 3D texture coordinates.

### Inputs and Outputs
*   **Inputs**: 3D volume data (scalar field), camera position (`eyePos`), view matrix (`mtx`), volume dimensions (`res`), and sampling rate (`sampleRate` or `numberOfSlices`).
*   **Outputs**: 2D image with color and opacity composited via alpha blending.

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
            billboardCenter[x] = eyePos[x] + i \cdot (-mtx[3x]) \quad \text{for } x=0,1,2
            $$
            Using `mtx[2]`, `mtx[6]`, `mtx[10]` for the view direction components [S3, unofficial].
        *   Construct a polygon primitive (triangle fan or quad) centered at `billboardCenter` with `size` [S3, S1, unofficial].
        *   Assign 3D texture coordinates to vertices. The coordinates map the unit quad to the volume bounds. In some implementations, a rotation matrix transforms texture coordinates in the vertex shader to align with the view, keeping the slice geometry stationary while changing the texture content [S4, unofficial].
        *   Store vertices in `a_position` and record `[offset, length]` in `offsets_length` [S2, unofficial].

3.  **Rendering**:
    *   Enable blending:
        ```python
        glEnable(GL_BLEND)
        glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
        ```
        [S3, G, unofficial].
    *   Bind vertex buffer and enable vertex attribute array for `a_position` [S2, unofficial].
    *   Iterate through `offsets_length`:
        *   Set vertex pointer:
            ```python
            glVertexAttribPointer(
                location,
                size=3,
                type=GL_FLOAT,
                normalize=False,
                stride=0,
                offset=offset * 4
            )
            ```
            [S2, unofficial].
        *   Draw slice:
            ```python
            glDrawArrays(GL_TRIANGLES, 0, length)
            ```
            [S2, unofficial].
    *   Fragment Shader:
        *   Sample `tex3D` at interpolated 3D texture coordinates to get scalar value [G].
        *   Map scalar value to color and opacity using `tex1D` (transfer function) [S4, unofficial].
        *   Composite using alpha blending [G].

### Parameters
*   `numberOfSlices` / `sampleRate`: Determines the number of slices. Higher values increase quality but reduce performance [S3, S4, unofficial].
*   `nearClip`, `farClip`: Orthographic projection planes (default 200.0, 600.0) [S3, unofficial].
*   `viewBound`: Half-width of the orthographic projection box (default 50) [S3, unofficial].
*   `boundary`: Integer flag for shader boundary checks (default 0) [S3, unofficial].

### Edge Cases and Pitfalls
*   **Texture Artifacts**: Using `GL_CLAMP_TO_BORDER` is critical when transforming texture coordinates to avoid sampling artifacts at slice edges [S4, S3, unofficial].
*   **Index Wrapping**: When constructing triangles from slice vertices, ensure the index for the third vertex wraps correctly (e.g., to index 1) if `k + 1` exceeds the array length to form closed polygons [S2, unofficial].
*   **View Alignment**: The proxy geometry must be strictly view-aligned to ensure samples can be correctly read from the frame buffer if post-processing or compositing relies on buffer reads [S5].
*   **Performance**: Geometry regeneration every frame is computationally expensive. The number of slices is often automatically reduced during rotation to maintain interactive rates [S7, unofficial].

## Subtopics

### Object-Aligned Slicing
Object-aligned slicing divides the volume into slices along the principal axes (x, y, z) rather than the view direction [S1, unofficial]. It uses three stacks of 2D textures corresponding to these directions. The active stack is swapped based on the viewpoint's closest principal direction [S1, unofficial]. It is faster and simpler but suffers from 3x memory consumption, data replication, image popping artifacts during stack swaps, and inconsistent sampling distances as the camera moves [S1, unofficial].

### Arbitrary Slicing
Arbitrary slicing uses a plane as proxy geometry to sample arbitrary planes for removing depth complexity and exploring volumes [S5]. The plane is view-aligned (rotated to be parallel to the image plane) and rendered to the back buffer [S5]. Results can be displayed in a separate viewport or as a textured polygon in object space [S5]. This allows for interactive probing and slicing without regenerating the full volume geometry [S5].

### Ray-to-Gravity-Center (R-GC) Volume Slicing
R-GC is a volume slicing strategy where parallel cutting planes are oriented perpendicular to the straight line connecting the point of reflection to the gravity center of the volume domain [S6]. It is used to generate Layered Polygonal Area Lights (LPALs) for indirect illumination [S6]. Compared to Ray-of-Regular-Reflection (R-RR) slicing, R-GC ensures LPALs face properly to the point of reflection regardless of the regular reflection direction, correctly rendering the upper part of the reflected appearance when the reflection point is apart from the volume [S6].

### GPU Volume Rendering Implementation
This variant implements view-aligned slicing on the GPU with pre-integration [S7, unofficial]. It uses two levels of pre-integration: simplified (neglecting color attenuation, faster) for interactive transfer function editing, and full pre-integration for final rendering [S7, unofficial]. Volume shading can be enabled using a per-fragment Phong lighting model based on the gradient of the scalar field [S7, unofficial]. The number of slices is automatically reduced during rotation to guarantee interactive rates [S7, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of slices and $S$ is the number of fragments per slice (proportional to screen resolution). Geometry generation is $O(N)$ [S3, S7, unofficial].
*   **Memory Complexity**: $O(V)$ for the 3D volume texture, where $V$ is the volume resolution [S3, S7, unofficial]. Object-aligned variants use $3 \times$ memory [S1, unofficial].
*   **Performance**: Interactive rates are achievable with moderate slice counts. Performance degrades if slice count is too high or if geometry regeneration is not optimized [S7, unofficial].
*   **Strengths**: Consistent sampling distance, minimal aliasing artifacts compared to object-aligned methods, efficient use of GPU rasterization [S1, S4, unofficial].
*   **Limitations**: Requires geometry regeneration every frame (CPU-GPU communication overhead), less flexible than ray casting for complex lighting effects without additional passes [S2, unofficial][S6].
*   **When to use**: Interactive volume visualization where view alignment is critical for quality and performance is acceptable [S7, unofficial].
*   **When not to use**: When extreme lighting complexity (e.g., accurate indirect illumination) is required without additional techniques, or when memory bandwidth is severely constrained [S6][S1, unofficial].

## Background
View-aligned slicing emerged as a method to combine the quality of ray casting with the speed of rasterization. It was notably implemented in systems like "Scientific-visualization-Slicing-volume-rendering" [S3, unofficial] and "SliceBasedVolumeRendering" [S4, unofficial]. The technique is discussed in the context of proxy geometry for interactive volume visualization [S5] and in implementations of pre-integrated volume rendering [S7, unofficial]. It addresses the limitations of object-aligned slicing, such as popping artifacts and inconsistent sampling [S1, unofficial].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S3] extrovert7986/Scientific-visualization-Slicing-volume-rendering, https://github.com/extrovert7986/Scientific-visualization-Slicing-volume-rendering, fetched 2026-10-26 (unofficial)
[S4] abhishektrip/SliceBasedVolumeRendering, https://github.com/abhishektrip/SliceBasedVolumeRendering, fetched 2026-10-27 (unofficial)
[S5] Integrating Dynamic Deformations into Interactive Volume Visualization, https://doi.org/10.2312/vissym.eurovis06.219-226, fetched 2026-10-31
[S6] Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights, https://doi.org/10.1111/cgf.13851, fetched 2026-10-26
[S7] Especificação de funções de transferência unidimensionais e multidimensionais para visualização volumétrica direta, http://hdl.handle.net/10183/12077, fetched 2026-10-30 (unofficial)
