# 3D Texture Proxy Geometry

Also called: 3D Texture Proxy, View-Aligned Slices, View-Aligned 3D Texture Slicing, Proxy Planes with 3D Texture, Planos proxy, Texturas 3D

## Overview

3D Texture Proxy Geometry is a direct volume rendering method that represents a rectilinear volume as a stack of parallel planar polygons (slabs) textured with 3D texture maps, leveraging hardware-accelerated tri-linear interpolation to sample volume data at arbitrary locations [S2, unofficial][S3, unofficial]. It solves the problem of efficient hardware-accelerated volume rendering with directional lighting by pre-computing color and opacity per voxel and storing them in 3D textures, allowing the graphics pipeline to handle interpolation and compositing [S3, unofficial]. Unlike shear-warp methods which fix plane counts to volume resolution, this method allows the number of rendering planes to be adjusted at runtime independent of volume resolution, and unlike coherent projection, it supports perspective views without a time penalty, though perspective projection causes irregular sampling along rays [S3, unofficial][S4, unofficial].

## How it works

**Inputs and Outputs:**
*   **Input:** A rectilinear 3D volume dataset with dimensions $(n_x, n_y, n_z)$ and spacing $(\Delta x, \Delta y, \Delta z)$. A viewing matrix (model-view and projection matrices) defining the camera position and orientation. A transfer function mapping scalar values to color and opacity.
*   **Output:** A 2D image where voxels are composited back-to-front using alpha blending.

**Data Structures:**
1.  **3D Texture Object:** A single volumetric texture stored in GPU memory.
    *   **Format:** Typically `GL_LUMINANCE_ALPHA` or `GL_RGBA` with pre-classified color/opacity values, or `GL_R16UI`/`GL_RED_INTEGER` with `usampler3D` for raw scalar data [S2, unofficial][S7, unofficial].
    *   **Dimensions:** $(N_x, N_y, N_z)$, where $N_i$ is the smallest power of two such that $N_i \ge n_i$ [S5, unofficial].
    *   **Layout:** The texture stores the volume data. If using a single 3D texture, it occupies one contiguous block of texture memory [S7, unofficial].
2.  **Proxy Geometry:** A set of 2D polygons (quads) representing slices through the volume.
    *   **Vertices:** Each vertex has 3D world coordinates $(x, y, z)$ and associated 3D texture coordinates $(s, t, r)$ [S14].
    *   **Count:** $K$ slices, where $K$ is a user-defined parameter (typically $K \approx d/\Delta z$ for parallel projection, where $d$ is the volume diagonal) [S5, unofficial].

**Numbered Steps:**

1.  **Data Upload:**
    *   Allocate a 3D texture object with dimensions $(N_x, N_y, N_z)$ [S5, unofficial].
    *   Upload the volume data into the texture. If the data is not power-of-two, it is padded or scaled by the GPU [S19, unofficial].
    *   Set texture parameters: `GL_TEXTURE_MIN_FILTER` and `GL_TEXTURE_MAG_FILTER` to `GL_LINEAR` (tri-linear interpolation) [S4, unofficial][S7, unofficial].
    *   *Pitfall:* If using integer data (e.g., `unsigned short`), the sampler must be `usampler3D`. Using `sampler3D` results in all values being 0 [S7, unofficial].
    *   *Pitfall:* In `glTexImage3D`, the `depth` parameter corresponds to the first dimension of the array. Passing the wrong dimension causes index confusion [S7, unofficial].

2.  **Pre-classification (Optional but Common):**
    *   Apply the transfer function to the volume data on the CPU or via a GPU compute shader to generate color $(C)$ and opacity $(A)$ values for each voxel [S3, unofficial].
    *   Store these values in the 3D texture. This allows the fragment shader to simply sample the texture without complex transfer function lookups [S2, unofficial].

3.  **Proxy Geometry Generation (Per Frame):**
    *   Determine the viewing direction.
    *   Define a bounding cube centered on the volume origin with side length $d = \sqrt{L_x^2 + L_y^2 + L_z^2}$, where $L_i = N_i \Delta i$ [S5, unofficial].
    *   Generate $K$ planes perpendicular to the view direction, spaced evenly within the bounding cube [S2, unofficial].
    *   For each plane, compute the intersection polygon with the bounding cube. This results in a quad (or clipped polygon) [S4, unofficial].
    *   *Optimization:* The intersection calculation can be offloaded to a vertex shader by recognizing the 12 edges of the cube as 6 sets, finding the intersection point of each edge with the arbitrary plane [S4, unofficial].
    *   Compute the 3D texture coordinates $(s, t, r)$ for each vertex of the proxy polygons.

4.  **Texture Coordinate Transformation:**
    *   The mapping from world space $(x, y, z)$ to texture space $(s, t, r)$ must account for the volume's physical dimensions and the current view rotation.
    *   The transformation matrix $M$ is composed of:
        1.  **Translation ($T$):** Shifts the volume center to the origin of texture space. $T = \text{translate}(\frac{n_x}{2N_x}, \frac{n_y}{2N_y}, \frac{n_z}{2N_z})$ [S5, unofficial].
        2.  **Scaling ($S$):** Scales the volume to fit the $[0,1]$ texture range. $S = \text{scale}(\frac{1}{L_x}, \frac{1}{L_y}, \frac{1}{L_z})$ [S5, unofficial].
        3.  **Rotation ($R^{-1}$):** Rotates the texture coordinates to align with the view. $R$ is the client's rotation matrix. The inverse $R^{-1}$ is applied to the texture coordinates [S5, unofficial].
    *   The combined transformation is:
        $$ (s, t, r) = (x, y, z) R^{-1} D^{-1} S T $$
        where $D$ is a uniform scale by $d$ (often absorbed into the bounding box definition) [S5, unofficial].
    *   *Critical Order:* If the volume has non-uniform spacing ($\Delta x \neq \Delta y \neq \Delta z$), the scaling matrix $S$ is non-uniform. Non-uniform scaling does **not** commute with rotation. The correct order is **Rotation then Scaling** (i.e., $R^{-1}$ applied before $S$ in the matrix multiplication chain, or $S$ applied to the rotated coordinates). The wrong order leads to shearing distortion [S5, unofficial].
    *   *Center Misalignment:* If the volume does not occupy the entire 3D texture (due to power-of-two padding), the volume center does not coincide with the texture center. Translations must be added to the texture matrix to align the volume center with the polygon set center [S19, unofficial].

5.  **Rendering and Compositing:**
    *   Set up the graphics pipeline:
        *   Disable lighting and depth buffer writing (if compositing over other geometry) [S20].
        *   Enable Alpha Blending: `glBlendFunc(GL_ONE, GL_ONE_MINUS_SRC_ALPHA)` [S20].
        *   Bind the 3D texture to a texture unit [S20].
    *   Clip the rendered polygons to the volume boundaries using six programmable world-space clipping planes positioned at the faces of the volume [S5, unofficial]. This avoids rendering regions outside the volume.
    *   Render the proxy polygons from **back-to-front** relative to the camera [S8, unofficial].
    *   For each polygon, the GPU rasterizes the quad. The vertex shader interpolates the $(s, t, r)$ texture coordinates across the polygon surface [S14].
    *   The fragment shader samples the 3D texture using the interpolated $(s, t, r)$ coordinates. Tri-linear interpolation is performed by the hardware [S8, unofficial].
    *   The sampled color and opacity are blended into the framebuffer [S15, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization (Once)
def init_volume(volume_data, transfer_function):
    # 1. Determine dimensions
    nx, ny, nz = volume_data.shape
    Nx = next_power_of_two(nx)
    Ny = next_power_of_two(ny)
    Nz = next_power_of_two(nz)
    
    # 2. Create 3D Texture
    texture_id = glGenTextures(1)
    glBindTexture(GL_TEXTURE_3D, texture_id)
    
    # 3. Upload data (Pre-classified or raw)
    # If raw, use usampler3D in shader. If pre-classified, use sampler3D.
    glTexImage3D(GL_TEXTURE_3D, 0, GL_RGBA, Nx, Ny, Nz, 0, GL_RGBA, GL_FLOAT, volume_data)
    
    # 4. Set filters for tri-linear interpolation
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MIN_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_MAG_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE)
    glTexParameteri(GL_TEXTURE_3D, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE)
    
    return texture_id, Nx, Ny, Nz, nx, ny, nz

# Per Frame
def render_frame(texture_id, Nx, Ny, Nz, nx, ny, nz, view_matrix, num_slices):
    # 1. Compute Bounding Cube Diagonal
    # Assuming uniform spacing for simplicity, or use actual dimensions
    Lx = Nx * 1.0 # Normalized scale factor
    Ly = Ny * 1.0
    Lz = Nz * 1.0
    d = sqrt(Lx**2 + Ly**2 + Lz**2)
    
    # 2. Generate Proxy Geometry
    # Get view direction from view_matrix
    view_dir = normalize(-view_matrix[2][:3]) # Assuming column-major OpenGL
    
    # Define planes perpendicular to view_dir
    # Spacing: d / num_slices
    slice_spacing = d / num_slices
    
    polygons = []
    for i in range(num_slices):
        # Calculate plane position along view direction
        # Center of volume is at origin
        offset = -d/2 + i * slice_spacing
        plane_pos = view_dir * offset
        
        # Compute intersection of plane with bounding box [-d/2, d/2]^3
        # This results in a quad (or clipped polygon)
        quad_vertices = compute_plane_cube_intersection(plane_pos, view_dir, d)
        
        # Compute Texture Coordinates for each vertex
        # Transformation: (s,t,r) = (x,y,z) * R^-1 * S * T
        # R^-1 is the inverse rotation part of the view matrix
        # S is scaling by 1/Lx, 1/Ly, 1/Lz
        # T is translation by (nx/2Nx, ny/2Ny, nz/2Nz)
        
        for v in quad_vertices:
            x, y, z = v
            
            # Apply Translation T
            tx = x + nx / (2.0 * Nx)
            ty = y + ny / (2.0 * Ny)
            tz = z + nz / (2.0 * Nz)
            
            # Apply Scaling S (Non-uniform)
            sx = tx / Lx
            sy = ty / Ly
            sz = tz / Lz
            
            # Apply Rotation R^-1
            # R^-1 is the upper 3x3 of the view matrix (assuming no translation in view matrix for rotation)
            # Note: View matrix transforms world to camera. 
            # We need to rotate texture coords to align with view.
            # The standard approach uses the texture matrix to apply R^-1 * S * T
            # Here we compute the final (s,t,r) manually for clarity
            
            # Extract rotation matrix from view_matrix (inverse of camera rotation)
            R_inv = view_matrix[:3, :3].T # Transpose of rotation part
            
            # Rotate the scaled coordinates
            s = dot(R_inv[0], [sx, sy, sz])
            t = dot(R_inv[1], [sx, sy, sz])
            r = dot(R_inv[2], [sx, sy, sz])
            
            polygons.append((x, y, z, s, t, r))
            
    # 3. Render
    glEnable(GL_BLEND)
    glBlendFunc(GL_ONE, GL_ONE_MINUS_SRC_ALPHA)
    glDisable(GL_DEPTH_TEST) # Or disable depth write if compositing over geometry
    glDisable(GL_LIGHTING)
    
    glBindTexture(GL_TEXTURE_3D, texture_id)
    
    # Set up clipping planes to restrict rendering to volume bounds
    setup_clipping_planes(view_matrix, d)
    
    # Draw polygons back-to-front
    # Note: Polygons must be sorted by depth relative to camera
    polygons.sort(key=lambda p: dot(p[:3], view_matrix[2][:3]))
    
    for poly in polygons:
        draw_quad_with_texture_coords(poly)
        
    glDisable(GL_BLEND)
    glEnable(GL_DEPTH_TEST)
    glEnable(GL_LIGHTING)
```

**Parameters:**
*   **Number of Slices ($K$):** Trade-off between quality and performance. More slices yield higher quality but lower frame rates. Default is often $d/\Delta z$ for parallel projection [S5, unofficial].
*   **Texture Resolution:** Must be power of two. $N_x, N_y, N_z \ge n_x, n_y, n_z$ [S5, unofficial].
*   **Interpolation:** Tri-linear (`GL_LINEAR`) is standard. Nearest-neighbor (`GL_NEAREST`) is faster but lower quality [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Perspective Projection:** Sampling rate is not constant for all rays because the distance of sampling points varies with the angle between the slice polygon and the viewing ray. This causes visual artifacts in fly-throughs with large FOV [S6, unofficial].
*   **Non-Uniform Scaling:** If $\Delta x \neq \Delta y \neq \Delta z$, the order of rotation and scaling in the texture matrix is critical. Rotation must precede scaling to avoid shearing [S5, unofficial].
*   **Memory Limits:** Large volumes may exceed texture memory. Bricking (subdividing into smaller volumes) is required but reduces performance [S3, unofficial][S6, unofficial].
*   **No Early Ray Termination:** All voxels are rendered; unseen voxels cannot be skipped, leading to unnecessary computation [S3, unofficial].

## Subtopics

### [Tri-linear Interpolation via LRP](913-tri-linear-interpolation-via-lrp.md)
Tri-linear interpolation via LRP is a GPU-accelerated method for sampling volumetric data stored in 3D textures, estimating the scalar value at arbitrary floating-point coordinates by linearly interpolating the eight surrounding voxel values. It solves the problem of resampling discrete voxel grids into continuous space for volume rendering, providing smoother structural details than direct voxel rendering. It sets itself apart from higher-order methods by using only linear blending factors and from CPU-based raycasting by leveraging hardware rasterization to interpolate texture coordinates "for free" between vertices.

### [Multi-Volume Rendering with Multiple Textures](1568-multi-volume-rendering-with-multiple-textures.md)
This method renders multiple distinct volumetric classes simultaneously by storing each class in a separate 3D texture and combining them during fragment shader execution. It solves the problem of visualizing heterogeneous data where different channels require independent transfer functions and opacities. It sets itself apart from single-volume approaches by allowing independent control of each channel's appearance, and from out-of-core techniques by relying entirely on GPU memory residency for all active channels.

### [Proxy Geometry Compression](1917-proxy-geometry-compression.md)
Proxy Geometry Compression is a technique for storing volume data and proxy geometry locally in graphics hardware to avoid CPU-GPU bandwidth bottlenecks associated with immediate mode rendering. It sets itself apart from immediate-mode splatting by pre-loading compressed proxy geometry templates into GPU memory, leveraging the regularity of rectilinear volumes to achieve high compression ratios (e.g., 171:1). This allows the entire volume representation to reside in local graphics memory, enabling interactive rendering speeds that are significantly faster than sending each splat separately.

### [3D Texture Proxy Geometry — 3D Texture Proxy, View-Aligned Slices, 3D-Texture Volume Rendering, 3D Texture-Based Volume Rendering](2532-3d-texture-proxy-geometry-3d-texture-proxy-view-aligned-slic.md)
3D Texture Proxy Geometry is a direct volume rendering method that represents a rectilinear volume as a stack of parallel planar polygons (slabs) textured with 3D texture maps, leveraging hardware-accelerated tri-linear interpolation to sample volume data at arbitrary locations. It solves the problem of efficient hardware-accelerated volume rendering with directional lighting by pre-computing color and opacity per voxel and storing them in 3D textures, allowing the graphics pipeline to handle interpolation and compositing. Unlike shear-warp methods which fix plane counts to volume resolution, this method allows the number of rendering planes to be adjusted at runtime independent of volume resolution, and unlike coherent projection, it supports perspective views without a time penalty, though perspective projection causes irregular sampling along rays.

### [Sm2D (2D Volume Texture Experiment)](3150-sm2d-2d-volume-texture-experiment.md)
Sm2D is a two-dimensional test application implemented as a 2D ray tracer to investigate the applicability of a texture-based approach for smoke volumes. It simplifies the three-dimensional volume rendering problem into a one-dimensional viewport function where tetrahedra are represented by triangles. It sets itself apart from its siblings by serving as a diagnostic tool to demonstrate the precision limits and rotational artifacts inherent in texture-based approximations compared to explicit geometric calculations.

### [Dependent Texture Reads for Classification](5640-dependent-texture-reads-for-classification.md)
Dependent Texture Reads (DTR) is a hardware extension that uses the color components of a fragment to generate texture coordinates, replacing the original color with values looked up from a texture. In volume rendering, it serves as an efficient mechanism for multidimensional transfer function evaluation (classification) and shading by treating fragment colors as inputs to a lookup table. Unlike standard texture sampling where coordinates are explicit attributes, DTR derives coordinates dynamically from the fragment's RGB values, enabling arbitrary function evaluation with up to three variables.

## Operating characteristics

*   **Time Complexity:** Image-order rendering. Computational cost scales with the number of pixels in the output image and the number of proxy planes ($K$), independent of the number of geometric primitives in the scene [S16, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the 3D texture storage [S7, unofficial].
*   **Measured Performance:** Historically much faster than ray casting on hardware with 3D texture support (e.g., SGI InfiniteReality, NVIDIA GeForce 3) [S8, unofficial][S16, unofficial]. Modern hardware has closed the performance gap with ray casting [S16, unofficial].
*   **Strengths:**
    *   Constant Euclidean distance between slices along a light ray for parallel projection [S1, unofficial].
    *   No artifacts from viewing angles (unlike axis-aligned 2D textures) [S1, unofficial].
    *   Hardware-accelerated tri-linear interpolation provides good image quality [S2, unofficial].
    *   Supports arbitrary viewing angles without recomputing the entire volume structure [S5, unofficial].
*   **Limitations:**
    *   Cannot skip unseen voxels (no early ray termination) [S3, unofficial].
    *   Performance degrades linearly with the number of proxy planes [S2, unofficial].
    *   Irregular sampling in perspective projection causes artifacts [S6, unofficial].
    *   Limited by texture memory size; large volumes require bricking [S3, unofficial].
    *   Aliasing artifacts from abrupt changes in intensity [S16, unofficial].
*   **When to Use:** Real-time interactive rendering of reasonably sized volumes on hardware with 3D texture support [S16, unofficial].
*   **When Not to Use:** Large volumes exceeding texture memory, or when high-quality optical modeling (e.g., refraction) is required [S16, unofficial].

## Background

*   **Origin:** Popularized by Cabral et al. in 1994 for nonshaded volume rendering [S3, unofficial].
*   **History:** Examined by Cabral et al. on the SGI RealityEngine, the first hardware to implement 3D texture mapping [S4, unofficial]. 3D textures were available in OpenGL as an extension (EXT texture 3D) for a long time, included in OpenGL 1.2. The first consumer product with support was the ATI Radeon in spring 2000. NVIDIA GeForce 3 also provided hardware implementation [S6, unofficial].
*   **Usage:** Historically deployed in Silicon Graphics InfiniteReality workstations. Used in modern graphics hardware with 3D texture acceleration for real-time interactive rendering [S16, unofficial].

## Sources

[S1] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S2] Visualización de grandes volúmenes vía bricking y texturas 3D, https://core.ac.uk/works/79987515, fetched 2026-10-27 (unofficial)
[S3] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S4] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S5] NASA Technical Reports Server (NTRS) 19970022121: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970022121, fetched 2026-10-26 (unofficial)
[S6] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S7] upupming/volume-rendering, https://github.com/upupming/volume-rendering, fetched 2026-10-26 (unofficial)
[S8] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S9] 16.2.2 3D Texture Volume Rendering - OpenGL, https://www.opengl.org/archives/resources/code/samples/sig99/advanced99/notes/node299.html, fetched 2026-10-27
[S10] 13.2 3D Texture Volume Rendering - OpenGL, https://www.opengl.org/archives/resources/code/samples/advanced/advanced97/notes/node181.html, fetched 2026-10-27
[S11] 13.2 3D Texture Volume Rendering, https://web.tecgraf.puc-rio.br/~ismael/Cursos/Cidade_CG/labs/OpenGL/OpenGL_siggraph1998/node226.html, fetched 2026-10-27
[S12] 16.2.2 3D Texture Volume Rendering - bluevoid.com, http://bluevoid.com/opengl/sig00/advanced00/notes/node308.html, fetched 2026-10-27
[S13] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S14] Interactive Volume Rendering Using 3D Texture-Mapping ..., https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27
[S15] Interactive Volume Rendering On PC Hardware (PDF), http://gd.tuwien.ac.at/visual/OpenQVis/Publ.2000.5.pdf, fetched 2026-10-27 (unofficial)
[S16] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28 (unofficial)
[S17] Interactive Volume Rendering on Standard PC Graphics ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28
[S18] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S19] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S20] GPU Gems - 第39章基于纹理的体绘制技术[译], http://pkuwwt.github.io/scholarship/2013-12-04-gpugems-ch39-texture-based-volume-rendering/, fetched 2026-10-30
