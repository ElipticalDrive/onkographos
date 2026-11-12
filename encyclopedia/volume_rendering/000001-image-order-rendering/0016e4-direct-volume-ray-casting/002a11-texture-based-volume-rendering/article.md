# Texture-Based Volume Rendering

Also called: Geometry-based volume visualization, Proxy Geometry Rendering, Slice-based rendering, TBVR, Texture Based Ray Casting

## Overview
Texture-Based Volume Rendering (TBVR) is a direct volume rendering technique that accelerates image-order ray casting by decomposing the 3D volume into a set of 2D planar slices, treating them as proxy geometry primitives (polygons) that are rasterized by the GPU [S1, unofficial][S2, unofficial]. It solves the performance bottleneck of software-based ray casting by leveraging the graphics pipeline's texture sampling and hardware alpha blending to perform compositing in parallel [S1, unofficial][S7, unofficial]. Unlike pure ray marching which computes samples per pixel, TBVR computes samples per slice vertex, making it significantly faster on older hardware but historically prone to sampling artifacts when viewing angles deviate from slice normals [S1, unofficial][S3, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset (regular grid), a camera position and orientation, and a transfer function (mapping scalar values to color/opacity).
*   **Output:** A 2D color image composited in the frame buffer.

**Data Structures:**
1.  **Volume Texture:** The scalar data is stored in GPU texture memory.
    *   *3D Texture:* A single `sampler3D` (or equivalent) containing the entire volume. Dimensions match the volume grid $(D_x, D_y, D_z)$. Format is typically single-channel (e.g., `GL_RED`, `GL_R8`) with linear filtering [S2, unofficial][S5, unofficial].
    *   *2D Texture Stack (Legacy):* Three separate stacks of 2D textures aligned with the X, Y, and Z axes. This requires data replication [S2, unofficial][S3, book].
2.  **Transfer Function Texture:** A 1D texture (`sampler1D`) mapping scalar intensity to RGBA color/opacity [S3, book].
3.  **Proxy Geometry:** A set of polygons (quads or triangle fans) representing slices through the volume.
    *   *Vertices:* Defined by the intersection of slicing planes with the volume bounding box.
    *   *Texture Coordinates:* Each vertex is assigned 3D texture coordinates $(u, v, w)$ that map directly to the volume texture space [S2, unofficial][S3, book].

**Numbered Steps:**

1.  **Initialize:**
    *   Load volume data into CPU memory.
    *   Process data if necessary (e.g., compute gradients for lighting, downsample) [S3, book].
    *   Create the transfer function lookup table (texture) [S3, book].
    *   Upload volume data to GPU as a 3D texture (or 2D stack) [S2, unofficial].
    *   Compile fragment shaders [S3, book].

2.  **Update (Per Frame/View Change):**
    *   **Compute Proxy Geometry:**
        *   Define a series of parallel slicing planes orthogonal to the viewing direction (view-aligned) or aligned with principal axes (object-aligned) [S2, unofficial][S7, unofficial].
        *   Intersect these planes with the volume bounding box to determine polygon vertices [S2, unofficial].
        *   Sort vertices in counter-clockwise order around the polygon centroid to form a triangle fan or quad [S2, unofficial].
        *   Assign 3D texture coordinates to each vertex. For a view-aligned slice, these coordinates map the slice plane into the 3D texture space [S3, book].
    *   **Sort Slices:** Sort the proxy geometry polygons back-to-front relative to the camera [S2, unofficial][S3, book].

3.  **Draw (Rasterization and Compositing):**
    *   Set rendering state:
        *   Disable lighting and depth testing (or disable depth writing if blending with opaque geometry) [S3, book][S5, unofficial].
        *   Enable alpha blending. For the "Over" operator (standard alpha blending), set source factor to `GL_ONE` and destination factor to `GL_ONE_MINUS_SRC_ALPHA` [S2, unofficial][S3, book].
        *   Bind the volume texture and transfer function texture [S3, book].
    *   **Render Loop (for each sorted slice):**
        *   Draw the proxy geometry polygon.
        *   **Vertex Shader:** Interpolates the 3D texture coordinates across the polygon [S3, book].
        *   **Fragment Shader:**
            1.  Sample the volume texture at the interpolated 3D coordinates to get scalar value $v$ [S2, unofficial][S3, book].
            2.  Use $v$ as a texture coordinate to sample the 1D transfer function texture, obtaining color $C$ and opacity $\alpha$ [S3, book].
            3.  (Optional) Apply illumination (e.g., Phong shading) using gradients computed from neighboring volume samples [S2, unofficial][S5, unofficial].
            4.  Output the resulting color and opacity to the frame buffer, where hardware blending accumulates the result [S2, unofficial].

**Formulas:**

*   **Volume Rendering Integral (Discrete Approximation):**
    The color $C$ and opacity $\alpha$ for a ray segment are accumulated. In TBVR, this is handled by the hardware blend function. For the "Over" operator:
    $$ C_{out} = C_{in} + C_{src} \cdot \alpha_{src} \cdot (1 - \alpha_{in}) $$
    $$ \alpha_{out} = \alpha_{in} + \alpha_{src} \cdot (1 - \alpha_{in}) $$
    Where $C_{in}, \alpha_{in}$ are the current frame buffer values, and $C_{src}, \alpha_{src}$ are the fragment shader outputs [S2, unofficial][S3, book].

*   **Transfer Function Lookup:**
    $$ (C, \alpha) = \text{TF}(v) $$
    Where $v$ is the scalar value sampled from the volume texture [S3, book].

*   **Gradient Estimation (for Lighting):**
    Using central differences with step size $\delta$ (e.g., $1/256$ in normalized texture space):
    $$ \mathbf{N} \approx \frac{\text{Tex}(u+\delta, v, w) - \text{Tex}(u-\delta, v, w)}{2\delta} \mathbf{i} + \dots $$
    The normal $\mathbf{N}$ is normalized and used in the Phong shading model [S5, unofficial].

**Parameters:**
*   **Slice Spacing:** Controls the number of slices. Finer spacing increases accuracy but reduces frame rate [S5, unofficial].
*   **Threshold:** A scalar value below which fragments are discarded (e.g., background) [S5, unofficial].
*   **Alpha Scale:** Multiplier for fragment opacity to adjust visibility [S5, unofficial].
*   **Lighting Constants:** Ambient ($K_a$), Diffuse ($K_d$), Specular ($K_s$), Shininess, Light positions [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(position, model_matrix, view_matrix, projection_matrix):
    # Compute world position for texture coordinate mapping
    world_pos = model_matrix * vec4(position, 1.0)
    # Pass world position to fragment shader for texture lookup
    gl_Position = projection_matrix * view_matrix * vec4(position, 1.0)
    return world_pos

# Fragment Shader
def fragment_shader(tex_coords_3d, volume_tex, tf_tex, alpha_scale, threshold):
    # 1. Sample volume data
    scalar_value = texture(volume_tex, tex_coords_3d).r
    
    # 2. Discard background/low-value voxels
    if scalar_value < threshold:
        discard()
        
    # 3. Apply Transfer Function
    # tf_tex is a 1D texture mapping scalar -> RGBA
    tf_color = texture(tf_tex, vec2(scalar_value, 0.5)).rgba
    
    # 4. Optional: Illumination (Phong)
    # Compute normal via finite differences in volume texture
    delta = 1.0 / 256.0
    grad_x = texture(volume_tex, tex_coords_3d + vec3(delta, 0, 0)).r - \
             texture(volume_tex, tex_coords_3d - vec3(delta, 0, 0)).r
    grad_y = texture(volume_tex, tex_coords_3d + vec3(0, delta, 0)).r - \
             texture(volume_tex, tex_coords_3d - vec3(0, delta, 0)).r
    grad_z = texture(volume_tex, tex_coords_3d + vec3(0, 0, delta)).r - \
             texture(volume_tex, tex_coords_3d - vec3(0, 0, delta)).r
    normal = normalize(vec3(grad_x, grad_y, grad_z))
    
    # Simple diffuse lighting (assuming light_dir and view_dir are available)
    # color = tf_color.rgb * (ambient + diffuse * max(dot(normal, light_dir), 0))
    color = tf_color.rgb # Simplified: no lighting in basic TBVR
    
    # 5. Set Alpha
    final_alpha = tf_color.a * alpha_scale
    
    # 6. Output
    return vec4(color, final_alpha)

# Main Render Loop
def render(volume_tex, tf_tex, camera, volume_bounds):
    # 1. Setup Blending
    enable_blend()
    set_blend_func(GL_ONE, GL_ONE_MINUS_SRC_ALPHA) # Over operator
    disable_depth_write()
    
    # 2. Generate Slices
    slices = generate_slices(camera, volume_bounds)
    
    # 3. Sort Slices Back-to-Front
    slices.sort(key=lambda s: s.distance_to_camera, reverse=True)
    
    # 4. Draw Slices
    for slice in slices:
        draw_polygon(slice.vertices, slice.tex_coords, fragment_shader)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Trilinear interpolation in 3D textures can cause blurring if the volume resolution is low. [S1, unofficial]
*   **Boundary Conditions:** Texture coordinates must be clamped or wrapped correctly to avoid sampling outside the volume bounds. [S5, unofficial]
*   **Degenerate Input:** If the camera is parallel to a slice plane, the slice may degenerate to a line or point, causing rendering artifacts. [S2, unofficial]

## Subtopics

### 2D Slice Replication Approach
Stores volume as three stacks of 2D textures (X, Y, Z aligned). Proxy geometry is simple rectangles. **Pros:** Fast, bilinear interpolation. **Cons:** 3x memory usage, data replication, "popping" artifacts when switching stacks, intensity variations as camera moves. **When to use:** Legacy hardware, fixed viewpoints. [S2, unofficial][S3, book][S9]

### View-Aligned Slicing
Slices are generated orthogonal to the viewing direction. **Pros:** Consistent sampling distance, no popping artifacts, better image quality. **Cons:** Slower geometry generation, requires 3D textures. **When to use:** Interactive applications with arbitrary camera movement. [S2, unofficial][S3, book]

### Object-Aligned Slicing
Slices are aligned with the principal axes of the volume. **Pros:** Simple geometry, fast. **Cons:** Artifacts at oblique angles, requires 3D texture stacks or replication. **When to use:** Orthographic views, static scenes. [S2, unofficial][S6, unofficial]

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of slices and $M$ is the number of fragments per slice. The complexity is independent of the number of pixels, unlike ray marching. [S3, book]
*   **Memory Complexity:** $O(V)$ for 3D textures, where $V$ is the volume size. $O(3V)$ for 2D texture stacks. [S2, unofficial][S3, book]
*   **Measured Performance:** On modern GPUs, TBVR achieves interactive frame rates (30+ FPS) for volumes up to $512^3$ with 3D textures. 2D texture approaches are faster on older hardware but suffer from quality degradation. [S1, unofficial][S3, book]
*   **Strengths:** High performance on GPUs, simple implementation, leverages hardware blending. [S1, unofficial][S3, book]
*   **Limitations:** Sampling artifacts (stair-stepping, blurring) if slice spacing is too large or interpolation is poor. View-dependent artifacts in object-aligned modes. [S1, unofficial][S2, unofficial]
*   **When to use:** Interactive visualization, medical imaging, legacy hardware. [S1, unofficial][S5, unofficial]
*   **When not to use:** High-fidelity offline rendering, volumes with extreme anisotropy, or when view-independent sampling is critical. [S3, book]

## Background
Texture-Based Volume Rendering was developed in the late 1990s and early 2000s as GPUs began to support 3D textures and fragment shaders. Key papers include Krüger and Westermann (2003) on GPU acceleration, and the NVIDIA GPU Gems Chapter 39 (2004) which detailed the implementation. It evolved from 2D texture slice approaches to 3D texture view-aligned slicing to address artifacts and memory issues. [S2, unofficial][S3, book][S7, unofficial]

## Sources
[S1] Direct Volume Rendering Data, https://www.scribd.com/document/1050958395/Direct-Volume-Rendering-Data, fetched 2026-10-25 (unofficial)
[S2] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S3] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S4] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S5] DoctorTone/MedicalViz, https://github.com/DoctorTone/MedicalViz, fetched 2026-10-27 (unofficial)
[S6] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S7] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S8] Volume Rendering Lecture Nov-30-2005 (Spanish), https://www.ciroduran.com/personal2/wp-content/uploads/2009/07/clase-volumerendering.pdf, fetched 2026-10-29 (unofficial)
[S9] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
