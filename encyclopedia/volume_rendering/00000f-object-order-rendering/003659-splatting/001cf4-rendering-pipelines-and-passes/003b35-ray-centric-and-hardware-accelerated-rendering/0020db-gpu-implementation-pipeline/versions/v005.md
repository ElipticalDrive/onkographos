# GPU Implementation Pipeline
Also called: Single-pass GPU Raycasting, Volumetric Raycasting

## Overview
A GPU-accelerated rendering technique that traces rays from the camera through each pixel into a 3D volume to compute color and opacity, implemented in a single pass using fragment shaders [S2, unofficial]. It solves the problem of visualizing volumetric data directly on the GPU without generating intermediate polygonal meshes [S4]. It differs from isosurface extraction (e.g., [Marching Cubes](53-marching-cubes.md)) by preserving internal volume information [S1], and from splatting ([Splatting](10-splatting.md)) by avoiding sorting and blending overheads for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D volume texture (scalar field), a transfer function (mapping scalar intensity to color and opacity), camera parameters (view matrix, projection matrix, ray origin, ray direction), and rendering mode (Isosurface, Alpha blending, or MIP) [S2, unofficial].
*   **Outputs:** A 2D color buffer (framebuffer) containing the final rendered image [S2, unofficial].

**Data Structures:**
*   **Volume Texture:** A 3D texture object (e.g., `GL_TEXTURE_3D`) storing voxel intensities. Typically stored as `GL_R8` or `GL_R16` for single-channel scalar data [S2, unofficial].
*   **Jitter Texture:** A 2D texture storing pre-generated random values for anti-aliasing [S2, unofficial].
*   **Ray Structure:** Defined by an origin point and a direction vector [S2, unofficial].
*   **AABB (Axis-Aligned Bounding Box):** Defined by `top` and `bottom` corner coordinates to bound the volume [S2, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Create the volume data structure and generate noise if required [S2, unofficial].
    *   Compute the view matrix by translating along the Z-axis based on an exponential distance exponent: $T_z = -4.0 \cdot \exp(\frac{m\_distExp}{600.0})$ [S2, unofficial].
    *   Compute the Model-View-Projection (MVP) matrix using a perspective projection with field of view $m\_fov$, aspect ratio $\frac{scaled\_width()}{scaled\_height()}$, near plane $0.1$, and far plane $100.0$ [S2, unofficial].
    *   Compute the ray origin in world space by inverting the view matrix and transforming the origin $(0,0,0)$ [S2, unofficial].

2.  **Shader Setup:**
    *   Bind the selected shader program (Vertex and Fragment) [S2, unofficial].
    *   Set uniform values including: `ViewMatrix`, `ModelViewProjectionMatrix`, `NormalMatrix`, `aspect_ratio`, `focal_length`, `viewport_size`, `ray_origin`, `top`, `bottom`, `background_colour`, `light_position`, `material_colour`, `step_length`, `threshold`, `gamma`, `volume` (texture unit 0), and `jitter` (texture unit 1) [S2, unofficial].
    *   Clear the color buffer with the `background_colour` [S2, unofficial].

3.  **Per-Pixel Ray Casting (Fragment Shader):**
    *   **Ray Direction Calculation:** For each fragment at screen coordinates $(x, y)$, compute the ray direction in view space:
        $$
        \begin{aligned}
        ray\_direction_{xy} &= 2.0 \cdot \frac{(x, y)}{viewport\_size} - 1.0 \\
        ray\_direction_x &\leftarrow ray\_direction_x \cdot aspect\_ratio \\
        ray\_direction_z &\leftarrow -focal\_length \\
        ray\_direction_{world} &= (ViewMatrix^{-1} \cdot (ray\_direction_{xy}, 0, 0)).xyz
        \end{aligned}
        $$
        [S2, unofficial]
    *   **Ray-Box Intersection:** Compute the entry ($t_0$) and exit ($t_1$) distances of the ray intersecting the AABB bounding box using the slab method:
        $$
        \begin{aligned}
        direction\_inv &= 1.0 / ray\_direction \\
        t_{top} &= direction\_inv \cdot (box.top - ray.origin) \\
        t_{bottom} &= direction\_inv \cdot (box.bottom - ray.origin) \\
        t_{min} &= \min(t_{top}, t_{bottom}) \\
        t_0 &= \max(0.0, \max(t_{min}.x, t_{min}.y)) \\
        t_{max} &= \max(t_{top}, t_{bottom}) \\
        t_1 &= \min(t_{max}.x, t_{max}.y)
        \end{aligned}
        $$
        [S2, unofficial]
    *   **Ray Parameterization:** Convert world-space intersection points to texture coordinates (normalized 0-1):
        $$
        \begin{aligned}
        ray\_start &= \frac{ray.origin + ray.direction \cdot t_0 - bottom}{top - bottom} \\
        ray\_stop &= \frac{ray.origin + ray.direction \cdot t_1 - bottom}{top - bottom} \\
        ray\_vector &= ray\_stop - ray\_start \\
        ray\_length &= \|ray\_vector\| \\
        step\_vector &= step\_length \cdot \frac{ray\_vector}{ray\_length}
        \end{aligned}
        $$
        [S2, unofficial]
    *   **Jittering:** Apply random jitter to the start position to reduce banding artifacts:
        $$
        ray\_start \leftarrow ray\_start + step\_vector \cdot texture(jitter, \frac{gl\_FragCoord.xy}{viewport\_size}).r
        $$
        [S2, unofficial]
    *   **Ray Marching Loop:** Initialize `colour` to $(0,0,0,0)$. While `ray_length > 0` and `colour.a < 1.0`:
        1.  Sample the volume: $intensity = texture(volume, position).r$ [S2, unofficial].
        2.  Apply Transfer Function (e.g., simple exponential alpha):
            $$
            \alpha = \frac{\exp(intensity) - 1.0}{\exp(1.0) - 1.0}, \quad c = (intensity, intensity, intensity, \alpha)
            $$
            [S2, unofficial]
        3.  Front-to-Back Alpha Blending:
            $$
            \begin{aligned}
            colour.rgb &= c.a \cdot c.rgb + (1 - c.a) \cdot colour.a \cdot colour.rgb \\
            colour.a &= c.a + (1 - c.a) \cdot colour.a
            \end{aligned}
            $$
            [S2, unofficial]
        4.  Advance:
            $$
            \begin{aligned}
            ray\_length &\leftarrow ray\_length - step\_length \\
            position &\leftarrow position + step\_vector
            \end{aligned}
            $$
            [S2, unofficial]

4.  **Final Compositing:**
    *   Blend the accumulated color with the background:
        $$
        \begin{aligned}
        colour.rgb &= colour.a \cdot colour.rgb + (1 - colour.a) \cdot pow(background\_colour, vec3(gamma)).rgb \\
        colour.a &= 1.0
        \end{aligned}
        $$
        [S2, unofficial]
    *   Apply Gamma Correction:
        $$
        a\_colour.rgb = pow(colour.rgb, vec3(1.0 / gamma))
        $$
        [S2, unofficial]

**Parameters:**
*   `step_length`: The distance between samples along the ray. Typical values depend on volume resolution and desired quality; smaller values increase accuracy but reduce performance [S2, unofficial].
*   `gamma`: Gamma correction value for display. Typical value is 2.2 [S2, unofficial].
*   `focal_length`: Determines the field of view. [S2, unofficial]
*   `m_distExp`: Exponential distance exponent for camera translation [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Blocks:** In multi-resolution structures, rays originating in empty blocks may miss neighboring non-empty blocks, causing occlusion artifacts. The ray must be advanced to the next non-empty block if a sample point is located in an empty block [S1].
*   **Interpolation Artifacts:** Flat blocking multi-resolution data management can cause interpolation problems at block boundaries [S1].
*   **Precision:** Ray-box intersection calculations must handle cases where ray direction components are zero (handled by `1.0 / ray.direction` in GLSL, which results in infinity, correctly handled by min/max logic) [S2, unofficial].

## Subtopics

### [Volume Rendering with 3D Textures and Ray Marching](2584-volume-rendering-with-3d-textures-and-ray-marching.md)
This method performs object-order volume rendering by ray marching through a 3D texture stored in GPU memory, applying a transfer function to map scalar values to color and opacity, and compositing the result front-to-back. It solves the problem of visualizing volumetric data (such as medical DICOM series or synthetic density fields) directly on the GPU without generating intermediate polygonal meshes. It differs from isosurface extraction (e.g., [Marching Cubes](53-marching-cubes.md)) by preserving internal volume information, and from splatting ([Splatting](10-splatting.md)) by avoiding sorting and blending overheads for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G].

### [NVIDIA GeForce3 Volume Rendering Pipeline](5630-nvidia-geforce3-volume-rendering-pipeline.md)
The NVIDIA GeForce3 Volume Rendering Pipeline is a hardware-accelerated method for interactive volume rendering that utilizes per-fragment dependent texture reads and 4-way multi-texturing to eliminate the need for intermediate scratch buffers. It distinguishes itself from prior software-based or scratch-buffer-dependent approaches (such as the SGI Octane2 method) by performing texture lookups dynamically based on computed fragment attributes, thereby avoiding expensive memory copy operations. This approach trades the flexibility of arbitrary 3D transfer functions for performance gains, constraining the transfer function to a separable product of 2D and 1D components due to hardware limitations on dependent texture dimensions.

### [Real-Time Rendering Pipeline with Virtual Memory](8407-real-time-rendering-pipeline-with-virtual-memory.md)
This method implements a real-time rendering pipeline for 3D Gaussian Splatting that streams Gaussian data to GPU memory on demand using a virtual memory system. It solves the memory capacity bottleneck of large scenes by mapping logical page IDs to physical GPU memory pages, replacing unused pages based on a least-recently-used (LRU) policy. Unlike software-based tile-sorting approaches (e.g., Kerbl et al.), this method performs global distance sorting of Gaussians in a GLSL compute shader before splatting them via the hardware rasterizer.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and the number of samples per ray (ray length / step length). The single-pass nature means it executes once per frame per pixel [S2, unofficial].
*   **Memory Complexity:** Dominated by the storage of the 3D volume texture and the framebuffer. The volume texture size is $W \times H \times D$ voxels [S2, unofficial].
*   **Measured Performance:** On an Nvidia GeForce 8800 Ultra with 768 MB graphics texture memory, a 512^3 volume with 8.9:1 data reduction (SVR size 256^3, piecewise segment length 8 voxels) achieves global light integration (A) in 284-552 ms and first order scattering incremental refinements (B) in 62-233 ms. High frame rates (15-22 fps) are reached after light updates [S1].
*   **Strengths:** Single-pass execution allows for real-time interaction. No intermediate buffers required for standard alpha blending [S2, unofficial].
*   **Limitations:** Computationally heavy per pixel compared to splatting for dense volumes [G]. Performance degrades with large step lengths or high-resolution volumes without multi-resolution acceleration [S1].
*   **When to Use:** Interactive visualization of dense volumetric data where internal structure is important.
*   **When Not to Use:** Extremely large datasets without multi-resolution support, or when hardware does not support 3D textures or fragment shaders [S1][S2, unofficial].

## Background
The method originated from the need for interactive volume rendering on GPUs. Early implementations utilized fragment shaders to perform ray casting in a single pass, eliminating the need for complex multi-pass accumulation buffers used in earlier software-based or fixed-function pipeline approaches [S2, unofficial]. The approach was popularized by open-source implementations and blog posts detailing GPU-accelerated volumetric raycasting using GLSL and C++ [S2, unofficial]. Variations involving global light propagation and multi-resolution data structures were developed to handle illumination and large datasets, such as the work presented at VG 2008 [S1].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S2] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
