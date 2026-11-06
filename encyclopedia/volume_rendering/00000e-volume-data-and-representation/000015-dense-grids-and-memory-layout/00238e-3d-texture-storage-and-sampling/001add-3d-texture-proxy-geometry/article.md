# 3D Texture Proxy Geometry

Also called: 3D Texture Mapping for Volume Rendering, 3D Texture Proxy, 3D Texture Volume Rendering, 3D Texture-Based Volume Rendering, Direct Volume Rendering via 3D Textures, View-Aligned 3D Texture Slicing, View-aligned 3D textured slices

## Overview
3D Texture Proxy Geometry is a direct volume rendering method that represents a rectilinear volume as a stack of parallel planar polygons (slabs) textured with 3D texture maps, leveraging hardware-accelerated tri-linear interpolation to sample volume data at arbitrary locations [S2, S3, unofficial]. It solves the problem of efficient hardware-accelerated volume rendering with directional lighting by pre-computing color and opacity per voxel and storing them in 3D textures, allowing the graphics pipeline to handle interpolation and compositing [S3, unofficial]. Unlike shear-warp methods which fix plane counts to volume resolution, this method allows the number of rendering planes to be adjusted at runtime independent of volume resolution, and unlike coherent projection, it supports perspective views without a time penalty, though perspective projection causes irregular sampling along rays [S3, unofficial][S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Rectilinear volumetric data grid of size $n_x \times n_y \times n_z$ with spacing $\Delta x, \Delta y, \Delta z$. Pre-classified color and opacity values (or raw data for post-classification). Viewpoint position and orientation (view matrix).
*   **Output:** 2D image with alpha blending.

**Data Structures:**
1.  **3D Texture Object:** Stores the entire volume. Dimensions $N_x, N_y, N_z$ are the smallest powers of two such that $N_i \ge n_i$ [S4, unofficial]. Format is typically `GL_RGBA` or `GL_LUMINANCE_ALPHA` [S1, unofficial].
2.  **Proxy Geometry:** A set of $K$ polygons (quads or triangles) representing slices through the volume. Each polygon has vertices with world-space coordinates $(x,y,z)$ and associated 3D texture coordinates $(s,t,r)$ [S13].

**Number of Slices:**
The default number of planes $K$ is chosen as $d / \Delta z_{eff}$, where $d$ is the diagonal of the bounding cube, ensuring each data point is sampled by at least one plane when viewed straight on [S4, unofficial]. Typically, $K$ is set to 2–4 times the default for higher accuracy [S4, unofficial].

**Transformation and Texture Coordinates:**
The volume is centered at the origin. A bounding cube of side $d = \sqrt{(n_x \Delta x)^2 + (n_y \Delta y)^2 + (n_z \Delta z)^2}$ is defined [S4, unofficial].
Texture coordinates $(s,t,r)$ map world coordinates $(x,y,z)$ to the 3D texture space $[0,1]^3$.
In the initial orientation (volume axes aligned with world axes), the mapping is:
$$ s(x) = \frac{x + 0.5 n_x \Delta x}{N_x \Delta x} $$
$$ t(y) = \frac{y + 0.5 n_y \Delta y}{N_y \Delta y} $$
$$ r(z) = \frac{z + 0.5 n_z \Delta z}{N_z \Delta z} $$
[S4, unofficial]

For arbitrary viewing angles, the bounding cube remains stationary in screen space, and the texture coordinates of the slice vertices are rotated. The transformation from world space $(x,y,z)$ to texture space $(s,t,r)$ is:
$$ \begin{pmatrix} s \\ t \\ r \end{pmatrix} = R^{-1} D^{-1} S T \begin{pmatrix} x \\ y \\ z \end{pmatrix} $$
Where:
*   $T$ is translation by $(0.5 n_x/N_x, 0.5 n_y/N_y, 0.5 n_z/N_z)$ [S4, unofficial].
*   $S$ is non-uniform scaling by $(d/L_x, d/L_y, d/L_z)$ where $L_i = N_i \Delta_i$ [S4, unofficial].
*   $D$ is uniform scaling by $d$ [S4, unofficial].
*   $R$ is the client's rotation matrix [S4, unofficial].

**Critical Order of Operations:**
If the volume has non-uniform spacing ($\Delta x \neq \Delta y \neq \Delta z$), $S$ does not commute with $R$. The correct order is **Rotation then Scaling** (i.e., $R^{-1}$ applied before $S$ in the matrix multiplication chain relative to the vector, or $S R^{-1}$ if pre-multiplying). Applying scaling before rotation causes shearing distortion [S4, unofficial].

**Algorithm Steps:**
1.  **Upload Data:** Load volume data into a single 3D texture object on the GPU. Dimensions must be powers of two. If the volume does not fill the texture, center it and apply translation in the texture matrix [S18, unofficial].
2.  **Compute Proxy Geometry:**
    *   Determine the view direction.
    *   Generate $K$ planes perpendicular to the view direction, spaced to cover the volume bounding box.
    *   Intersect these planes with the volume bounding box (or use 6 world-space clipping planes to clip the rendered quads) [S4, unofficial].
    *   For each vertex of the resulting polygons, compute the 3D texture coordinates $(s,t,r)$ using the transformation matrix derived from the current view orientation [S13].
    *   *Optimization:* The intersection calculation can be offloaded to a vertex shader by recognizing the 12 edges of the cube as 6 sets, finding the intersection of each edge with the arbitrary plane [S3, unofficial].
3.  **Render Loop (Per Frame):**
    *   Update the texture matrix to reflect the current view rotation $R^{-1}$ [S4, unofficial].
    *   Sort the proxy polygons from back to front relative to the viewer [S7, unofficial].
    *   For each polygon:
        *   Rasterize the polygon.
        *   The GPU hardware trilinearly interpolates the $(s,t,r)$ coordinates across the polygon surface [S13].
        *   Perform a dependent texture lookup (or standard 3D texture fetch) using the interpolated $(s,t,r)$ to retrieve pre-classified color and opacity [S3, unofficial].
        *   If post-classification is used, apply the transfer function in the fragment shader using the interpolated scalar value [S19].
    *   Blend the fragments back-to-front using alpha blending: $C_{out} = C_{in} + C_{src}(1 - A_{in})$ [S19].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_3d_texture_proxy(volume_data, view_matrix, num_slices):
    # 1. Setup 3D Texture
    # Determine power-of-two dimensions
    Nx = next_power_of_two(volume_data.dim_x)
    Ny = next_power_of_two(volume_data.dim_y)
    Nz = next_power_of_two(volume_data.dim_z)
    
    # Create and upload 3D texture
    tex_id = create_3d_texture(Nx, Ny, Nz, volume_data.data)
    set_texture_filters(tex_id, GL_LINEAR, GL_LINEAR) # Trilinear interpolation
    
    # 2. Compute Bounding Cube and Transformation Matrices
    # Volume physical dimensions
    Lx = volume_data.dim_x * volume_data.spacing_x
    Ly = volume_data.dim_y * volume_data.spacing_y
    Lz = volume_data.dim_z * volume_data.spacing_z
    
    # Diagonal of the bounding cube
    d = sqrt(Lx**2 + Ly**2 + Lz**2)
    
    # Scaling factors for texture coordinates
    # Note: S scales by d/Li to map physical size to normalized texture space scaled by d
    S = Matrix4x4.scale(d/Lx, d/Ly, d/Lz)
    
    # Translation to center volume in texture space [0,1]
    # Offset is 0.5 * (voxel_count / texture_dim)
    Tx = 0.5 * volume_data.dim_x / Nx
    Ty = 0.5 * volume_data.dim_y / Ny
    Tz = 0.5 * volume_data.dim_z / Nz
    T = Matrix4x4.translate(Tx, Ty, Tz)
    
    # Uniform scale by diagonal
    D = Matrix4x4.scale(d, d, d)
    
    # 3. Generate Proxy Geometry
    # Get inverse rotation from view matrix
    R_inv = view_matrix.rotation().inverse()
    
    # Combined texture matrix: (s,t,r) = R^-1 * D^-1 * S * T * (x,y,z)
    # Note: D^-1 is 1/d. Since D commutes with R, we can simplify or keep order.
    # Correct order for non-uniform spacing: Rotation then Scaling.
    texture_matrix = R_inv * D.inverse() * S * T
    
    # Generate K planes perpendicular to view direction
    # Plane spacing approx d / num_slices
    plane_spacing = d / num_slices
    
    polygons = []
    for i in range(num_slices):
        # Define a plane at distance z' from center along view axis
        # Intersect plane with bounding box of size d
        # Result is a polygon (quad or clipped quad)
        poly = intersect_plane_with_cube(plane_spacing * (i - num_slices/2), view_matrix, d)
        
        # Compute texture coordinates for each vertex
        for vertex in poly.vertices:
            # Transform world coordinate to texture coordinate
            tex_coord = texture_matrix * vertex.world_pos
            vertex.tex_coord = tex_coord
            
        polygons.append(poly)
        
    # 4. Render
    enable_alpha_blending()
    disable_depth_write() # Or depth test only if needed for sorting
    
    # Sort polygons back-to-front
    polygons.sort(key=lambda p: distance_to_camera(p.center))
    
    bind_texture(tex_id)
    set_texture_matrix(texture_matrix)
    
    for poly in polygons:
        draw_polygon(poly)
        
    disable_alpha_blending()
```

## Subtopics

### [Tri-linear Interpolation via LRP](913-tri-linear-interpolation-via-lrp.md)
Tri-linear interpolation via LRP is a GPU-accelerated method for estimating scalar values at arbitrary floating-point coordinates within a 3D volume grid by interpolating the eight surrounding voxel values. It solves the problem of sampling continuous volume data from discrete voxel grids, providing smooth rendering compared to direct voxel rendering which exhibits less structural detail. It is distinguished from other interpolation methods by its specific implementation using the GPU's LRP (linear interpolation) instruction, leveraging hardware rasterization to interpolate texture coordinates "for free" between vertices, thereby optimizing memory bandwidth usage.

### [Multi-Volume Rendering with Multiple Textures](1568-multi-volume-rendering-with-multiple-textures.md)
This method renders multiple distinct volumetric classes simultaneously by storing each class in a separate 3D texture and combining them during fragment shader execution. It solves the problem of visualizing heterogeneous data where different channels require independent transfer functions and opacities. It sets itself apart from single-volume approaches by allowing independent control of each channel's appearance, and from out-of-core techniques by relying entirely on GPU memory residency for all active channels.

### [Proxy Geometry Compression](1917-proxy-geometry-compression.md)
Proxy Geometry Compression is a technique for storing volume data and proxy geometry locally in graphics hardware to avoid CPU-GPU bandwidth bottlenecks associated with immediate mode rendering. It sets itself apart from immediate-mode splatting by pre-loading compressed proxy geometry templates into GPU memory, leveraging the regularity of rectilinear volumes to achieve high compression ratios (e.g., 171:1). This allows the entire volume representation to reside in local graphics memory, enabling interactive rendering speeds that are significantly faster than sending each splat separately.

### [Sm2D (2D Volume Texture Experiment)](3150-sm2d-2d-volume-texture-experiment.md)
Sm2D is a 2-dimensional test application implemented as a 2D ray tracer to investigate the applicability of a texture-based approach for smoke volumes. It simplifies the 3D volume rendering problem into a 1D viewport function where tetrahedra are represented by triangles. It sets itself apart from its siblings by serving as a diagnostic tool to demonstrate the precision limits and rotational artifacts inherent in texture-based approximations compared to explicit geometric calculations.

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Requires $O(N_x N_y N_z)$ texture memory, where $N_i$ are power-of-two dimensions [S6, unofficial]. This is often larger than the raw data size. Large volumes may exceed texture memory, requiring partitioning into subvolumes (bricking) which introduces swapping overhead [S2, unofficial].
*   **Time:** Computational cost scales with the number of pixels in the output image (image-order rendering), independent of the number of geometric primitives [S15, unofficial]. The cost is dominated by rasterization and texture fetches.

**Measured Performance:**
*   Historically provided tremendous speedup over ray casting on hardware like SGI RealityEngine and later NVIDIA GeForce 3 [S7, unofficial][S15, unofficial].
*   Modern hardware has closed the performance gap with ray casting, but 3D texture proxy geometry often yields lower image quality due to aliasing and limited optical modeling [S15, unofficial].
*   Optimization techniques like occlusion clipping in fragment programs can yield an average 2.8x speedup [S3, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Constant Euclidean distance between slices along a light ray for parallel projection; no artifacts from viewing angles; good image quality due to trilinear interpolation; flexible slice orientation [S1, unofficial][S6, unofficial].
*   **Limitations:** Cannot skip unseen voxels (unlike ray casting); perspective projection causes irregular sampling rates leading to visual artifacts in fly-throughs [S2, unofficial][S5, unofficial]; limited by texture memory size; requires recomputation of proxy geometry on every view change [S16]; non-uniform spacing requires careful matrix ordering to avoid shearing [S4, unofficial].

**When to Use:**
*   Real-time interactive rendering of reasonably sized volumes on hardware with 3D texture support [S15, unofficial].
*   Scenarios requiring directional lighting and pre-classification [S4, unofficial].
*   When constant sampling rate in parallel projection is required [S5, unofficial].

**When Not to Use:**
*   Very large volumes that exceed texture memory without efficient bricking strategies [S2, unofficial].
*   Applications requiring complex optical models (e.g., refraction) or accurate ray skipping [S15, unofficial].
*   Perspective fly-throughs with large field-of-view angles where sampling artifacts are unacceptable [S5, unofficial].

## Background
3D Texture Proxy Geometry was popularized by Cabral et al. in 1994 for nonshaded volume rendering [S2, unofficial][S8]. It was examined on the SGI RealityEngine, the first hardware to implement 3D texture mapping [S3, unofficial]. The method treats the volume rendering integral as a generalized Radon transform [S3, unofficial]. 3D textures were available in OpenGL as an extension (EXT texture 3D) for a long time before being included in OpenGL 1.2 [S5, unofficial]. The first consumer product with support was the ATI Radeon in spring 2000, followed by NVIDIA GeForce 3 [S5, unofficial]. Historically deployed in Silicon Graphics InfiniteReality workstations, it is now a standard feature on modern GPUs [S15, unofficial].

## Sources
[S1] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S2] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S3] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S4] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
[S5] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S6] upupming/volume-rendering, https://github.com/upupming/volume-rendering, fetched 2026-10-26 (unofficial)
[S7] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S8] 16.2.2 3D Texture Volume Rendering - OpenGL, https://www.opengl.org/archives/resources/code/samples/sig99/advanced99/notes/node299.html, fetched 2026-10-27
[S9] 13.2 3D Texture Volume Rendering - OpenGL, https://www.opengl.org/archives/resources/code/samples/advanced/advanced97/notes/node181.html, fetched 2026-10-27
[S10] 13.2 3D Texture Volume Rendering, https://web.tecgraf.puc-rio.br/~ismael/Cursos/Cidade_CG/labs/OpenGL/OpenGL_siggraph1998/node226.html, fetched 2026-10-27
[S11] 16.2.2 3D Texture Volume Rendering - bluevoid.com, http://bluevoid.com/opengl/sig00/advanced00/notes/node308.html, fetched 2026-10-27
[S12] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S13] Interactive Volume Rendering Using 3D Texture-Mapping ..., https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27
[S14] Interactive Volume Rendering On PC Hardware (PDF), http://gd.tuwien.ac.at/visual/OpenQVis/Publ.2000.5.pdf, fetched 2026-10-27 (unofficial)
[S15] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28 (unofficial)
[S16] Interactive Volume Rendering on Standard PC Graphics ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
[S17] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S18] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S19] GPU Gems - 第39章基于纹理的体绘制技术[译], http://pkuwwt.github.io/scholarship/2013-12-04-gpugems-ch39-texture-based-volume-rendering/, fetched 2026-10-30
[S20] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
