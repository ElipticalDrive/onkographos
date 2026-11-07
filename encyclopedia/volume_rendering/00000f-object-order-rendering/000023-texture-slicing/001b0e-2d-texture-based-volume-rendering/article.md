# 2D Texture-Based Volume Rendering

Also called: 2D Texture Mapping, 2D Texture Volume Rendering, Axis-Aligned Slice Rendering, Stack of 2D textures, VR Basado en Texturas 2D

## Overview
2D Texture-Based Volume Rendering is an object-order technique that renders volumetric data by decomposing the 3D scalar field into a set of axis-aligned 2D slices, storing each slice as a 2D texture, and compositing them back-to-front using hardware alpha blending [S2, S3, S4, S7, unofficial][S19][S20, unofficial]. It solves the problem of rendering volumetric data on hardware that lacks 3D texture support or to exploit the high performance of 2D texture mapping units [S2, S7, unofficial][S14, S15, S19]. Unlike view-aligned 3D texture methods which require trilinear interpolation and per-frame geometry regeneration, this method uses object-aligned slices and bilinear interpolation, trading mathematical accuracy for compatibility with older graphics hardware and reduced geometry generation overhead [S2, S4, S7, S9, S20, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V$ of dimensions $D_x \times D_y \times D_z$. A viewing direction vector $\vec{v}$. A transfer function $TF$ mapping scalar values to RGBA.
*   **Output:** A 2D image buffer $I$ of size $H \times W$.

**Data Structures:**
The volume is decomposed into three stacks of 2D textures, one for each major axis (X, Y, Z).
*   **Texture Stack X:** Contains $D_x$ textures, each of size $D_y \times D_z$. Each texture represents a slice perpendicular to the X-axis.
*   **Texture Stack Y:** Contains $D_y$ textures, each of size $D_x \times D_z$. Each texture represents a slice perpendicular to the Y-axis.
*   **Texture Stack Z:** Contains $D_z$ textures, each of size $D_x \times D_y$. Each texture represents a slice perpendicular to the Z-axis.
*   **Texel Format:** RGBA. The RGB components store the emission coefficient $I_{emit}$ (color), and the Alpha component stores the inverse absorption coefficient $(1 - \vartheta_k)$ [S4, unofficial].
*   **Proxy Geometry:** For each selected slice, a polygon (typically a quad or triangle fan) representing the intersection of the slice plane with the volume bounding box is generated [S6, S7, S20, unofficial].

**Steps:**

1.  **Preprocessing (Offline/Init):**
    *   Decompose the volume $V$ into three sets of 2D textures [S2, unofficial][S14, S15, S16, S17, S19][S20, unofficial].
    *   Apply the transfer function to map scalar values to RGBA. Store these in the 2D textures [S4, S11, S12, unofficial].
    *   *Note:* If the volume dimensions are not powers of two, pad the 2D slices to the next power of two (e.g., 181x217x181 padded to 256x256 per slice) [S20, unofficial].

2.  **View Selection (Per Frame):**
    *   Extract the rotation component from the ModelView matrix [S4, unofficial].
    *   Transform the initial viewing direction vector $\vec{v}_{init} = (0, 0, -1)$ by the rotation matrix to get $\vec{v}_{obj}$ in object space [S4, unofficial].
    *   Find the component of $\vec{v}_{obj}$ with the largest absolute value: $n_{max} = \text{argmax}_i |v_{obj,i}|$ [S4, unofficial].
    *   Select the texture stack corresponding to the axis $n_{max}$. If $v_{obj, n_{max}} > 0$, render from the positive end; otherwise, from the negative end [S4, unofficial].
    *   *Constraint:* The chosen stack's normal must be within $\pm 45^\circ$ of the viewing direction to avoid seeing through slice edges [S7, unofficial][S10, S14, S15, S16, S17].

3.  **Geometry Generation:**
    *   For each slice in the selected stack, compute the intersection polygon with the volume bounding box [S6, S7, S9, unofficial][S14, S15, S16, S17][S20, unofficial].
    *   Generate proxy geometry (quads or triangle fans) for these intersections [S6, S7, S20, unofficial].
    *   Assign texture coordinates to the vertices of each polygon corresponding to the position within the 2D texture [S9, unofficial][S14, S15, S16, S17].

4.  **Rendering Loop:**
    *   Sort the slices in back-to-front order relative to the camera [S2, S3, S4, S5, S7, unofficial][S10, S14, S15, S16, S17][S20, unofficial].
    *   For each slice $k$ from back to front:
        a. Bind the corresponding 2D texture.
        b. Render the proxy geometry.
        c. The fragment shader samples the texture. Bilinear interpolation is used within the slice [S3, S4, unofficial][S19].
        d. Composite the fragment color $C_{src}$ and opacity $A_{src}$ into the frame buffer using alpha blending [S2, S3, S4, S5, S7, unofficial][S10, S14, S15, S16, S17][S20, unofficial].

**Formulas:**

*   **Radiative Transfer Equation (Discrete):**
    $$ I(s_k) = I(s_{k-1}) \cdot \vartheta_k + I_{emit}(s_k) \cdot (1 - \vartheta_k) $$
    where $I(s_k)$ is the radiance at sample $k$, $\vartheta_k$ is the absorption coefficient, and $I_{emit}(s_k)$ is the emission coefficient [S4, unofficial].

*   **Alpha Blending Equation:**
    $$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
    where $C_{src} = I_{emit}$ and $A_{src} = (1 - \vartheta_k)$ [S4, unofficial]. This implements the discrete volume rendering integral when slices are processed in correct depth order [S3, unofficial].

*   **Slice Count (Optional/Advanced):**
    Some implementations use a formula to determine the number of slices $N$ based on max planes and quality factor $q$:
    $$ N = 2 \cdot (10 + \text{max\_planes} \cdot q^3) $$
    [S6, unofficial].

**Parameters:**
*   **Number of Slices:** Typically matches the texel dimensions of the volume data cube along the slicing axis [S15, S16, S17].
*   **Opacity Correction:** Optional scaling of slice alpha to compensate for varying Euclidean distance between slices along a light ray depending on viewing angle [S2, S6, unofficial]. Default is often OFF [S6, unofficial].
*   **Texture Format:** GL_LUMINANCE or GL_RGBA. GL_LUMINANCE stores opacity only, requiring multiplication by tissue color in the shader [S20, unofficial]. GL_RGBA stores pre-computed color and opacity [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **45-Degree View Artifacts:** When the viewing direction is close to $45^\circ$ relative to the slice normal, the viewer sees through the edges of the slices, causing visible gaps and artifacts [S2, S7, unofficial][S10, S14, S15, S16, S17].
*   **Stack Switching:** Abrupt switching between texture stacks (X, Y, Z) causes visible popping artifacts due to changes in interpolation artifacts and memory swap times [S4, S20, unofficial].
*   **Bilinear Interpolation Artifacts:** Since 2D textures only support bilinear interpolation, the third dimension is not interpolated, leading to severe aliasing and lack of spatial smoothness, especially when zooming [S3, S18, unofficial][S19][S20, unofficial].
*   **Off-Origin Volume:** If the volume bounding box is offset from the origin, standard VBO shift-scale mechanisms may misalign texture coordinates, reading from density 0 corners. Disabling shift-scale or correcting texcoords is required [S6, unofficial].
*   **Negative Values:** Applying logarithmic transfer functions to data with zero or negative values results in NaN or -inf, causing rendering errors [S11, S12, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_2d_texture_volume(volume_data, view_transform, image_buffer):
    """
    Renders a 3D volume using 2D texture-based slicing.
    
    Args:
        volume_data: 3D numpy array (Dx, Dy, Dz)
        view_transform: 4x4 OpenGL ModelView matrix
        image_buffer: 3D numpy array (H, W, 3) initialized to 0
        
    Returns:
        image_buffer: Composited RGB image
    """
    
    # 1. Preprocessing: Decompose volume into 3 stacks of 2D textures
    # Stack X: Dx textures of size (Dy, Dz)
    # Stack Y: Dy textures of size (Dx, Dz)
    # Stack Z: Dz textures of size (Dx, Dy)
    # Each texel stores RGBA (Color, Opacity)
    stacks = decompose_volume_to_2d_textures(volume_data)
    
    # 2. Select Slicing Axis
    # Extract rotation from ModelView matrix
    rotation_matrix = extract_rotation(view_transform)
    # Initial view direction in camera space (0, 0, -1)
    view_dir_cam = np.array([0.0, 0.0, -1.0])
    # Transform to object space
    view_dir_obj = rotation_matrix @ view_dir_cam
    
    # Find axis with maximum absolute component
    abs_view = np.abs(view_dir_obj)
    axis_idx = np.argmax(abs_view)
    
    # Determine direction (positive or negative along axis)
    direction = 1 if view_dir_obj[axis_idx] > 0 else -1
    
    # Select the appropriate stack
    if axis_idx == 0: # X-axis
        stack = stacks['X']
        slice_dims = (volume_data.shape[1], volume_data.shape[2])
    elif axis_idx == 1: # Y-axis
        stack = stacks['Y']
        slice_dims = (volume_data.shape[0], volume_data.shape[2])
    else: # Z-axis
        stack = stacks['Z']
        slice_dims = (volume_data.shape[0], volume_data.shape[1])
        
    # 3. Generate Proxy Geometry and Sort Slices
    # For each slice in the stack, compute intersection with volume bounding box
    # Return list of (slice_index, polygon_vertices, texture_coords)
    slices = generate_slice_polygons(stack, volume_data.shape, view_transform)
    
    # Sort slices back-to-front relative to camera
    # This requires transforming polygon centers to view space and sorting by depth
    slices = sort_slices_back_to_front(slices, view_transform)
    
    # 4. Render Slices
    for slice_info in slices:
        slice_idx = slice_info.index
        polygon = slice_info.polygon
        tex_coords = slice_info.tex_coords
        
        # Bind the 2D texture for this slice
        texture = stack[slice_idx]
        
        # Render polygon with texture
        # The fragment shader samples the texture using bilinear interpolation
        # and composites the result into the frame buffer
        render_slice(polygon, tex_coords, texture, image_buffer)
        
    return image_buffer

def render_slice(polygon, tex_coords, texture, image_buffer):
    """
    Renders a single slice onto the image buffer using alpha blending.
    """
    # Rasterize the polygon
    fragments = rasterize(polygon, tex_coords)
    
    for frag in fragments:
        # Sample texture (Bilinear Interpolation)
        color, opacity = texture_sample(texture, frag.uv)
        
        # Alpha Blending: C_dest = C_src * A_src + C_dest * (1 - A_src)
        # Assuming image_buffer stores current composite color
        current_color = image_buffer[frag.y, frag.x]
        new_color = color * opacity + current_color * (1 - opacity)
        image_buffer[frag.y, frag.x] = new_color
```

## Subtopics

### [Novel Sampling Scheme for Slice Based Volume Rendering](953-novel-sampling-scheme-for-slice-based-volume-rendering.md)
A volume rendering technique that generates proxy geometry using multiple object-aligned 2D texture stacks clipped by planes extending from the front-facing edges of the volume's bounding box in the viewing direction, eliminating stack-switching artifacts and jagged boundary artifacts. Unlike traditional 2D texture-based volume rendering which selects a single stack causing visible color/opacity shifts during rotation, this method renders multiple clipped stacks simultaneously. It approximates the uniform sampling pattern of ray casting more closely than spherical shell approaches while avoiding their large geometry overhead.

### [Interpolation of Arbitrary Slices (Multi-Texture Blending)](2089-interpolation-of-arbitrary-slices-multi-texture-blending.md)
This method renders arbitrary slicing planes through a volume by interpolating between two adjacent object-aligned texture slices using multi-texturing hardware. It solves the problem of rendering non-axis-aligned cross-sections without requiring 3D texture storage or complex ray-casting, by decomposing the arbitrary slice into polygon strips that span only two 2D textures. It sets itself apart from single-texture approaches by reducing the polygon count by half, though it is constrained by memory bandwidth limitations.

### [Multi-Texture Interpolation](2200-multi-texture-interpolation.md)
Multi-Texture Interpolation is a technique for 2D texture-based volume rendering that eliminates visual artifacts caused by a fixed number of discrete slices. It enables real trilinear interpolation by computing intermediate slices on the fly using multi-texture hardware combiners. Unlike standard fixed-slice approaches that produce staircase or banding artifacts, this method retains the standard slice projection and enhances the interpolation step between adjacent slices using GPU combiners.

### [Slice-Based Volume Rendering with HSV Transfer Function](4329-slice-based-volume-rendering-with-hsv-transfer-function.md)
Slice-Based Volume Rendering with HSV Transfer Function is a GPU-accelerated object-order technique that renders volumetric data by projecting a series of 2D slices onto the image plane and blending them back-to-front, while mapping scalar values to color using a Hue-Saturation-Value (HSV) transfer function. It differs from standard 2D texture-based volume rendering by explicitly managing a texture array (sprite sheet) to simulate 3D trilinear filtering and by computing color in HSV space rather than using a pre-baked 1D RGB texture lookup. It differs from its sibling [Novel Sampling Scheme for Slice Based Volume Rendering](953-novel-sampling-scheme-for-slice-based-volume-rendering.md) by relying on fixed sprite-sheet indexing rather than a novel sampling distribution, and from [Interpolation of Arbitrary Slices (Multi-Texture Blending)](2089-interpolation-of-arbitrary-slices-multi-texture-blending.md) by using a single texture array with calculated intra-slice coordinates instead of blending multiple distinct texture units.

### SGI Octane2 Volume Rendering Pipeline
A rendering pipeline for the SGI Octane2 with V series graphics cards that uses 3D pixel textures via `glDrawPixels()` or `glCopyPixels()` to handle multi-dimensional transfer functions and shading [S22]. It renders slices from the VGH data volume and pixel textures them, then renders a 'Shade' slice and modulates it with the classified slice, blending into the frame buffer [S22]. When the volume is rotated, it updates the shade volume once the rotation is completed to maintain interactive efficiency [S22]. It generates a new quantized normal pixel texture for shading, renders each slice of the quantized normal volume orthographically in the scratch buffer, copies the slice from the scratch buffer to the corresponding slice in the shade volume, and re-renders the volume with the updated shade volume [S22]. Unlike the GeForce3, this method requires a scratch/auxiliary buffer because pixel texture does not occur directly on a per-fragment basis during rasterization [S22]. Updating the shade volume in hardware requires that the quantized normal slices are always smaller than the scratch buffer’s dimensions [S22].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot H \cdot W)$, where $N$ is the number of slices and $H \times W$ is the image resolution. The cost is dominated by rasterization and blending of $N$ polygons [S7, S9, unofficial].
*   **Memory Complexity:** $O(D_x \cdot D_y \cdot D_z)$ for storing three full stacks of 2D textures. This is significantly higher than 3D texture methods which store the volume once [S4, S5, S20, unofficial]. For a $181 \times 217 \times 181$ volume padded to $256 \times 256$, memory usage can exceed 37 MB for the texture stacks alone [S20, unofficial].
*   **Performance:** Historically much faster than ray casting, but the performance gap has closed with modern hardware [S9, unofficial]. Near-interactive frame rates (e.g., 1.6 fps for a Blunt-fin dataset) were achieved on older hardware like Pentium-IV with QuadroFX 3400 [S1, unofficial]. Performance depends on image size, not scene complexity [S7, S9, unofficial].
*   **Strengths:** Works on older hardware without 3D texture support [S2, S7, unofficial][S14, S15]. Fast due to hardware bilinear interpolation [S2, S7, unofficial]. Simple implementation [S3, unofficial].
*   **Limitations:** Lower image quality than ray casting or 3D texture methods due to lack of trilinear interpolation [S3, S9, S18, unofficial][S19]. Visible artifacts at oblique angles (45 degrees) [S2, S7, unofficial]. Stack switching artifacts [S4, S20, unofficial]. High memory consumption [S4, S5, unofficial].
*   **When to Use:** Legacy hardware, real-time applications where speed is prioritized over quality, or when 3D texture support is unavailable.
*   **When Not to Use:** High-quality medical visualization requiring smooth interpolation, or when memory is constrained.

## Background
The basic idea of using object-aligned slices to substitute trilinear by bilinear interpolation was presented by Lacroute and Levoy [S19], although the original implementation did not use texturing hardware. The method was studied by researchers after 2D texture mapping became available on GPU, with Engel et al. providing an excellent introduction [S3, unofficial]. It was used by companies such as Silicon Graphics in their InfiniteReality system [S9, unofficial]. Supported by many modern graphics hardware devices, it remains a foundational technique in volume rendering history [S9, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S3] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S4] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S5] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S6] cvc::volslice: port VolumeRover2's view-aligned slice volume renderer, https://github.com/transfix/libcvc/pull/332, fetched 2026-10-26 (unofficial)
[S7] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S8] Volume Rendering in OpenGL using 2D textures, https://stackoverflow.com/questions/44370174/volume-rendering-in-opengl-using-2d-textures, fetched 2026-10-27 (unofficial)
[S9] Volume Ray Casting Techniques and Applications using ..., https://halogenica.net/bin/graphics/volume-rendering-using-gpgpu/Thesis.pdf, fetched 2026-10-27 (unofficial)
[S10] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S11] SREESUBBARAYUDU/Parallel_Volume_Rendering, https://github.com/SREESUBBARAYUDU/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S12] overhaul88/Parallel_Volume_Rendering, https://github.com/overhaul88/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S13] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
[S14] 13.3 2D Texture Volume, https://www.opengl.org/archives/resources/code/samples/advanced/advanced97/notes/node182.html, fetched 2026-10-27
[S15] 13.3 2D Texture Volume, https://web.tecgraf.puc-rio.br/~ismael/Cursos/Cidade_CG/labs/OpenGL/OpenGL_siggraph1998/node227.html, fetched 2026-10-27
[S16] 16.2.3 2D Texture Volume, http://bluevoid.com/opengl/sig00/advanced00/notes/node309.html, fetched 2026-10-27
[S17] 16.2.3 2D Texture Volume, https://www.opengl.org/archives/resources/code/samples/sig99/advanced99/notes/node300.html, fetched 2026-10-27
[S18] Interactive Volume Rendering On PC Hardware (PDF), http://gd.tuwien.ac.at/visual/OpenQVis/Publ.2000.5.pdf, fetched 2026-10-27 (unofficial)
[S19] Interactive Volume Rendering on Standard PC Graphics ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
[S20] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S21] NodeVolumeTextureRender (MIPAV API Documentation), https://mipav.cit.nih.gov/documentation/api/gov/nih/mipav/view/renderer/J3D/surfaceview/NodeVolumeTextureRender.html, fetched 2026-10-30
[S22] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
