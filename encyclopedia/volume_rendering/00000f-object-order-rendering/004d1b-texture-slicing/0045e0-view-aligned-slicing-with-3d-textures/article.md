# View-Aligned Slicing with 3D Textures

Also called: View-Aligned 3D Texture Slicing, 3D-Texture Based Slicing, View-aligned slice rendering

## Overview
View-aligned slicing with 3D textures is a direct volume rendering method that represents a volumetric dataset as a single 3D texture object, intersected by a stack of parallel proxy planes oriented perpendicular to the viewer's line of sight [S1, S3, S5, S6, unofficial][S10][S16][S19][S21, unofficial][S22, unofficial]. It solves the sampling artifacts and orientation limitations of axis-aligned 2D texture slicing by maintaining a constant Euclidean distance between slices along any light ray, regardless of viewing angle [S1, S6, unofficial][S7][S16][S24, unofficial]. Unlike ray casting, it offloads integration and interpolation to hardware rasterization, enabling interactive frame rates for moderate-sized volumes, though it cannot skip unseen voxels and is limited by texture memory capacity [S3, unofficial][S10, S19].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset (voxel grid), a transfer function (mapping scalar values to color/opacity), and a camera configuration (view direction, projection type) [S19][S21, unofficial].
*   **Output:** A 2D image composited from back-to-front blended slices.

**Data Structures:**
1.  **3D Texture Object:** Stores the entire volume data. Dimensions $(N_x, N_y, N_z)$ must be powers of two, at least as large as the voxel counts $(n_x, n_y, n_z)$ [S5, unofficial][S17, unofficial][S24, unofficial].
    *   *Format:* Typically `GL_RED` (single channel scalar) or `GL_COLOR_INDEX8_EXT` (paletted) [S15][S1, unofficial].
    *   *Layout:* Voxel $(i, j, k)$ maps to texture coordinate $( (i + 0.5 n_x A_x)/(N_x A_x), (j + 0.5 n_y A_y)/(N_y A_y), (k + 0.5 n_z A_z)/(N_z A_z) )$ [S5, unofficial].
2.  **Proxy Geometry:** A set of $M$ parallel polygons (quads or clipped polygons) representing slices [S19][S21, unofficial].
    *   *Vertices:* Each vertex has a 3D position in world space and corresponding 3D texture coordinates $(s, t, r)$ [S17, unofficial][S21, unofficial].
    *   *Clipping:* Polygons are clipped against the volume's bounding box to exclude empty space [S21, unofficial][S25].

**Numbered Steps:**

1.  **Volume Upload:** Load the scalar volume data into GPU memory as a 3D texture [S14, S18]. If the data is not power-of-two, pad with zeros and scale via hardware [S16][S24, unofficial]. Set minification/magnification filters to linear interpolation, which enables hardware trilinear interpolation [S4, unofficial][S19].
2.  **View Extraction:** Retrieve the current model-view matrix [S20, unofficial]. Extract the view direction (eye vector), right vector, and up vector from the matrix columns [S22, unofficial].
3.  **Slice Generation:** Compute $M$ slice planes perpendicular to the view direction, spaced evenly through the volume's bounding box depth [S2, S21, unofficial].
    *   Determine the depth range $[z_{min}, z_{max}]$ of the volume along the view axis [S8, unofficial].
    *   For each slice index $i \in [0, M-1]$, calculate the slice position:
        $$ z_i = z_{min} + \frac{(z_{max} - z_{min}) \cdot (i + 0.5)}{M} $$
    *   Construct a quad centered at the volume center, oriented orthogonal to the view, with size sufficient to contain the volume's diagonal [S20, S24, unofficial].
    *   Clip the quad against the volume's 6 bounding planes to form the final polygon vertices [S5, S21, unofficial].
4.  **Texture Coordinate Calculation:** For each vertex of the proxy geometry, compute its 3D texture coordinates $(s, t, r)$ [S17, unofficial].
    *   Let $(x, y, z)$ be the world-space position of the vertex [S5, unofficial].
    *   Let $(n_x, n_y, n_z)$ be the voxel counts, $(A_x, A_y, A_z)$ be voxel spacings, and $(N_x, N_y, N_z)$ be texture dimensions [S5, unofficial].
    *   Let $R$ be the rotation matrix aligning the volume's local axes to the world axes (often identity if volume is centered at origin and aligned) [S5, unofficial].
    *   The transformation from world space $(x,y,z)$ to texture space $(s,t,r)$ is:
        $$ \begin{bmatrix} s \\ t \\ r \end{bmatrix} = R^{-1} \begin{bmatrix} \frac{x + 0.5 n_x A_x}{N_x A_x} \\ \frac{y + 0.5 n_y A_y}{N_y A_y} \\ \frac{z + 0.5 n_z A_z}{N_z A_z} \end{bmatrix} $$
        *Note: If the volume is rotated relative to the texture axes, $R^{-1}$ must be applied. Uniform scaling commutes with rotation, but non-uniform scaling (due to anisotropic voxels) does not; scaling must occur after rotation in the texture matrix pipeline [S5, unofficial].*
5.  **Rendering Loop (Back-to-Front):**
    *   Sort the $M$ slices by their depth $z_i$ relative to the camera (farthest to nearest) [S2, S3, unofficial][S10].
    *   For each slice:
        a.  Bind the 3D texture.
        b.  Rasterize the proxy polygon. The GPU interpolates the $(s, t, r)$ texture coordinates across the polygon [S17, unofficial].
        c.  In the fragment shader, sample the 3D texture:
            $$ \text{color\_val} = \text{texture3D}(u\_Volume, \vec{tex\_coord}) $$
        d.  Apply the transfer function to map $\text{color\_val}$ to emission color $C$ and opacity $A$ [S19][S25].
        e.  Composite using alpha blending:
            $$ C_{dest} = C_{src} \cdot A_{src} + C_{dest} \cdot (1 - A_{src}) $$
            $$ A_{dest} = A_{src} + A_{dest} \cdot (1 - A_{src}) $$
            (Standard over operator) [S3, unofficial][S10].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def next_power_of_two(v):
    p = 1
    while p < v:
        p *= 2
    return p

def create_3d_texture(width, height, depth, data):
    # Allocates GPU memory and copies data, padding with zeros if dimensions are smaller than powers of two.
    return {"width": width, "height": height, "depth": depth, "data": data}

def set_filter(texture, mag_filter, min_filter):
    # Configures hardware interpolation. GL_LINEAR enables trilinear interpolation for 3D textures.
    texture["mag_filter"] = mag_filter
    texture["min_filter"] = min_filter

def enable_blend():
    # Enables hardware alpha blending.
    pass

def set_blend_func(src_factor, dst_factor):
    # Sets the blending equation: dest = src * src_factor + dest * dst_factor.
    pass

def disable_depth_write():
    # Disables writing to the depth buffer to allow correct back-to-front compositing.
    pass

def bind_texture(texture, unit):
    # Binds the texture to the specified texture unit for sampling.
    pass

def clip_quad_against_bbox(vertices, bbox_min, bbox_max):
    # Intersects the quad plane with the 6 faces of the bounding box.
    # Returns a list of vertices forming the clipped polygon (3 to 6 vertices).
    clipped = []
    for i in range(len(vertices)):
        v_curr = vertices[i]
        v_next = vertices[(i + 1) % len(vertices)]
        inside_curr = all(v_curr[k] >= bbox_min[k] and v_curr[k] <= bbox_max[k] for k in range(3))
        inside_next = all(v_next[k] >= bbox_min[k] and v_next[k] <= bbox_max[k] for k in range(3))
        if inside_curr:
            clipped.append(v_curr)
        if inside_curr != inside_next:
            t = 0.0
            for k in range(3):
                if abs(v_next[k] - v_curr[k]) > 1e-6:
                    if v_curr[k] < bbox_min[k]:
                        t = max(t, (bbox_min[k] - v_curr[k]) / (v_next[k] - v_curr[k]))
                    elif v_curr[k] > bbox_max[k]:
                        t = max(t, (bbox_max[k] - v_curr[k]) / (v_next[k] - v_curr[k]))
            clipped.append(vec3(v_curr[k] + t * (v_next[k] - v_curr[k]) for k in range(3)))
    return clipped

def draw_polygon(vertices, tex_coords):
    # Rasterizes the polygon, interpolates tex_coords, samples the bound 3D texture,
    # applies the transfer function, and blends the result into the framebuffer.
    pass

def init_volume(volume_data, transfer_function):
    nx, ny, nz = volume_data.shape
    Nx, Ny, Nz = next_power_of_two(nx), next_power_of_two(ny), next_power_of_two(nz)
    gpu_texture = create_3d_texture(Nx, Ny, Nz, volume_data)
    set_filter(gpu_texture, "GL_LINEAR", "GL_LINEAR")
    return gpu_texture, (Nx, Ny, Nz), (nx, ny, nz)

def render_view_aligned_slices(gpu_texture, volume_dims, voxel_counts, camera, num_slices=64):
    mv_matrix = camera.get_modelview_matrix()
    right = normalize(mv_matrix[0:3:4])
    up = normalize(mv_matrix[1:3:4])
    eye = normalize(mv_matrix[2:3:4])
    
    half_extents = [c / 2.0 for c in voxel_counts]
    bbox_min = vec3(-half_extents[0], -half_extents[1], -half_extents[2])
    bbox_max = vec3(half_extents[0], half_extents[1], half_extents[2])
    
    center = vec3(0, 0, 0)
    min_z = dot(bbox_min, eye)
    max_z = dot(bbox_max, eye)
    
    slices = []
    diag = length(bbox_max - bbox_min)
    quad_size = diag / 2.0
    
    for i in range(num_slices):
        t = (i + 0.5) / num_slices
        z_pos = min_z + t * (max_z - min_z)
        slice_center = center + eye * z_pos
        v0 = slice_center + (-right - up) * quad_size
        v1 = slice_center + (right - up) * quad_size
        v2 = slice_center + (right + up) * quad_size
        v3 = slice_center + (-right + up) * quad_size
        
        clipped_verts = clip_quad_against_bbox([v0, v1, v2, v3], bbox_min, bbox_max)
        
        if len(clipped_verts) > 0:
            tex_coords = []
            for v in clipped_verts:
                sx = (v.x + 0.5 * voxel_counts[0]) / volume_dims[0]
                sy = (v.y + 0.5 * voxel_counts[1]) / volume_dims[1]
                sz = (v.z + 0.5 * voxel_counts[2]) / volume_dims[2]
                tex_coords.append(vec3(sx, sy, sz))
            slices.append({'depth': z_pos, 'vertices': clipped_verts, 'tex_coords': tex_coords})
    
    slices.sort(key=lambda s: s['depth'], reverse=True)
    
    enable_blend()
    set_blend_func("GL_SRC_ALPHA", "GL_ONE_MINUS_SRC_ALPHA")
    disable_depth_write()
    bind_texture(gpu_texture, unit=0)
    
    for slice in slices:
        draw_polygon(slice['vertices'], slice['tex_coords'])
```

**Parameters:**
*   `num_slices`: Number of proxy planes. Default is often $d/A_z$ (diagonal length divided by voxel depth spacing) or matched to the smallest texture dimension [S5, unofficial][S12]. Higher values improve quality but reduce performance [S2, unofficial].
*   `sliceDistance`: Controls spacing between slices. Inverse of sampling rate [S21, unofficial].
*   `quad_size`: Half-width of the proxy quad. Must be at least half the volume diagonal to prevent "popping" artifacts when the volume rotates [S20, S24, unofficial].

**Edge Cases and Pitfalls:**
*   **Anisotropic Voxels:** If voxel spacings $A_x, A_y, A_z$ differ, the scaling matrix $S$ does not commute with rotation $R$. The texture matrix must apply rotation *before* non-uniform scaling to avoid shearing distortion [S5, unofficial].
*   **Perspective Projection:** The sampling rate is not constant for all rays because the distance between sampling points varies with the angle between the slice polygon and the viewing ray. This causes artifacts in fly-throughs with large field-of-view angles [S6, unofficial][S12].
*   **Texture Memory Limits:** The entire volume must fit in GPU memory. Large volumes require "bricking" (subdivision) or out-of-core swapping, which adds complexity and latency [S3, unofficial][S7][S22, unofficial].
*   **Center Misalignment:** If the volume data does not fill the entire power-of-two 3D texture, the volume center may not coincide with the texture center. A translation must be added to the texture matrix to align them [S24, unofficial].
*   **Data Type Mismatch:** Using `sampler3D` for unsigned short data results in zero values; `usampler3D` must be used [S7].

## Subtopics

### [Tri-linear Interpolation via LRP](913-tri-linear-interpolation-via-lrp.md)
Tri-linear interpolation via LRP is a GPU-accelerated method for sampling volumetric data stored in 3D textures, estimating the scalar value at arbitrary floating-point coordinates by linearly interpolating the eight surrounding voxel values. It solves the problem of resampling discrete voxel grids into continuous space for volume rendering, providing smoother structural details than direct voxel rendering. It sets itself apart from higher-order methods by using only linear blending factors and from CPU-based raycasting by leveraging hardware rasterization to interpolate texture coordinates "for free" between vertices.

### [Multi-Volume Rendering with Multiple Textures](1568-multi-volume-rendering-with-multiple-textures.md)
This method renders multiple distinct volumetric classes simultaneously by storing each class in a separate 3D texture and combining them during fragment shader execution. It solves the problem of visualizing heterogeneous data where different channels require independent transfer functions and opacities. It sets itself apart from single-volume approaches by allowing independent control of each channel's appearance, and from out-of-core techniques by relying entirely on GPU memory residency for all active channels.

### [Proxy Geometry Compression](1917-proxy-geometry-compression.md)
Proxy Geometry Compression is a technique for storing volume data and proxy geometry locally in graphics hardware to avoid CPU-GPU bandwidth bottlenecks associated with immediate mode rendering. It sets itself apart from immediate-mode splatting by pre-loading compressed proxy geometry templates into GPU memory, leveraging the regularity of rectilinear volumes to achieve high compression ratios (e.g., 171:1). This allows the entire volume representation to reside in local graphics memory, enabling interactive rendering speeds that are significantly faster than sending each splat separately.

### [3D Texture Proxy Geometry — 3D Texture Proxy, View-Aligned Slices, 3D-Texture Volume Rendering, 3D Texture-Based Volume Rendering](2532-3d-texture-proxy-geometry-3d-texture-proxy-view-aligned-slic.md)
3D Texture Proxy Geometry is a direct volume rendering technique that represents a volumetric dataset as a stack of parallel planar polygons (slabs) textured with a 3D texture map, leveraging hardware-accelerated tri-linear interpolation to resample the volume at arbitrary locations. It solves the performance bottleneck of ray-casting by offloading the integration and interpolation steps to the graphics pipeline's texture mapping hardware, enabling interactive rates for moderate-sized volumes. Unlike shear-warp methods which fix the number of planes to the volume resolution, this method allows the number of proxy planes to be adjusted at runtime independent of the data size, offering a trade-off between image quality and frame rate.

### [Sm2D (2D Volume Texture Experiment)](3150-sm2d-2d-volume-texture-experiment.md)
Sm2D is a two-dimensional test application implemented as a 2D ray tracer to investigate the applicability of a texture-based approach for smoke volumes. It simplifies the three-dimensional volume rendering problem into a one-dimensional viewport function where tetrahedra are represented by triangles. It sets itself apart from its siblings by serving as a diagnostic tool to demonstrate the precision limits and rotational artifacts inherent in texture-based approximations compared to explicit geometric calculations.

### [Dependent Texture Reads for Classification](5640-dependent-texture-reads-for-classification.md)
Dependent Texture Reads (DTR) is a hardware extension that uses the color components of a fragment to generate texture coordinates, replacing the original color with values looked up from a texture. In volume rendering, it serves as an efficient mechanism for multidimensional transfer function evaluation (classification) and shading by treating fragment colors as inputs to a lookup table. Unlike standard texture sampling where coordinates are explicit attributes, DTR derives coordinates dynamically from the fragment's RGB values, enabling arbitrary function evaluation with up to three variables.

### Paletted 3D Texture Rendering
A rendering technique using a 3D texture where the index is identical to the scalar value, pointing to a color/opacity palette (transfer function) applied during runtime [S1, unofficial]. It involves creating a 3D texture with format `GL_COLOR_INDEX8_EXT` and defining a 1D transfer function lookup table using `glColorTableEXT` [S1, unofficial]. The palette size is typically 256 entries, each 4 bytes (RGBA) [S1, unofficial]. This allows simple and fast changes to the transfer function at runtime, unlike pre-classified textures which require complete redefinition of texture data [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Image-order rendering. Computational cost scales with the number of pixels in the output image and the number of proxy planes $M$, independent of the scene object count [S19][S21, unofficial]. Performance degrades linearly as the number of proxy planes increases [S2, unofficial].
*   **Memory Complexity:** Requires $O(N_x N_y N_z)$ texture memory for the volume data [S7][S17, unofficial]. Additional memory for proxy geometry is minimal [S21, unofficial].
*   **Measured Performance:** Historically provided tremendous speedup over ray casting on hardware like SGI RealityEngine and modern GPUs due to high rasterization performance [S10, S19]. However, performance is specific to hardware capabilities (3D texture support) [S10].
*   **Strengths:** Constant Euclidean distance between slices along light rays; no orientation artifacts; good image quality with trilinear interpolation; simpler implementation than ray casting; allows arbitrary viewing angles [S1, S6, unofficial][S7][S16].
*   **Limitations:** Cannot skip unseen voxels (all voxels are rendered) [S3, unofficial]; limited by texture memory size (requires bricking for large volumes) [S3, unofficial][S17, unofficial]; aliasing artifacts from abrupt intensity changes [S19]; inconsistent sampling rate in perspective projection [S6, unofficial]; difficulty implementing complex optical models (e.g., refraction) [S19].
*   **When to Use:** Real-time interactive visualization of moderate-sized volumes (e.g., medical imaging, scientific data) where hardware supports 3D textures and high frame rates are required [S19][S22, unofficial].
*   **When Not to Use:** Very large volumes exceeding texture memory without bricking; applications requiring adaptive sampling or complex light transport (refraction, scattering); perspective fly-throughs with large FOV where sampling artifacts are unacceptable [S6, unofficial][S19].

## Background
The method was popularized by Cabral et al. in 1994 for nonshaded volume rendering on the SGI RealityEngine, the first hardware to implement 3D texture mapping [S3, S4, unofficial][S22, unofficial]. It was examined in detail by Cabral et al. [5] and Rezk-Salama et al. [50] proposed GPU vertex shader optimizations. 3D textures were available in OpenGL as an extension (EXT texture 3D) for a long time, included in OpenGL 1.2, with the first consumer product support being the ATI Radeon in spring 2000 and NVIDIA GeForce 3 [S6, unofficial][S19]. It was historically used by Silicon Graphics in their InfiniteReality workstations and remains a standard technique in modern graphics hardware for real-time volume rendering [S19][S22, unofficial].

## Sources
[S1] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S2] Visualización de grandes volúmenes vía bricking y texturas 3D, https://core.ac.uk/works/79987515, fetched 2026-10-27 (unofficial)
[S3] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S4] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S5] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
[S6] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S7] upupming/volume-rendering, https://github.com/upupming/volume-rendering, fetched 2026-10-26
[S8] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S9] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26 (unofficial)
[S10] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26
[S11] 16.2.2 3D Texture Volume Rendering - OpenGL, https://www.opengl.org/archives/resources/code/samples/sig99/advanced99/notes/node299.html, fetched 2026-10-27 (unofficial)
[S12] 13.2 3D Texture Volume Rendering - OpenGL, https://www.opengl.org/archives/resources/code/samples/advanced/advanced97/notes/node181.html, fetched 2026-10-27
[S13] 13.2 3D Texture Volume Rendering, https://web.tecgraf.puc-rio.br/~ismael/Cursos/Cidade_CG/labs/OpenGL/OpenGL_siggraph1998/node226.html, fetched 2026-10-27
[S14] 16.2.2 3D Texture Volume Rendering - bluevoid.com, http://bluevoid.com/opengl/sig00/advanced00/notes/node308.html, fetched 2026-10-27
[S15] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27
[S16] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S17] Interactive Volume Rendering Using 3D Texture-Mapping ..., https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27 (unofficial)
[S18] Interactive Volume Rendering On PC Hardware (PDF), http://gd.tuwien.ac.at/visual/OpenQVis/Publ.2000.5.pdf, fetched 2026-10-27
[S19] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28
[S20] view aligned slices - OpenGL: Advanced Coding, https://community.khronos.org/t/view-aligned-slices/38447, fetched 2026-10-28 (unofficial)
[S21] instantreality 1.0 - tutorial - Volume Rendering, https://doc.instantreality.org/tutorial/volume-rendering/, fetched 2026-10-28 (unofficial)
[S22] Interactive Volume Rendering on Standard PC Graphics ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28 (unofficial)
[S23] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29 (unofficial)
[S24] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S25] GPU Gems - 第39章基于纹理的体绘制技术[译], http://pkuwwt.github.io/scholarship/2013-12-04-gpugems-ch39-texture-based-volume-rendering/, fetched 2026-10-30
