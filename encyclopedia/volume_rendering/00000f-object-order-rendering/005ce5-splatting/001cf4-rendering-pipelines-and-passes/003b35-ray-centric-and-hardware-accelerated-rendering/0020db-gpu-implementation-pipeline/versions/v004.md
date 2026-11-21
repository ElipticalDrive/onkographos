# GPU Implementation Pipeline

Also called: Single-pass GPU Raycasting, Volumetric Raycasting

## Overview
A GPU-accelerated rendering technique that traces rays from the camera through each pixel into a 3D volume to compute color and opacity, implemented in a single pass using fragment shaders [S2, unofficial]. It solves the problem of visualizing volumetric data directly on the GPU without generating intermediate polygonal meshes [S4]. It differs from isosurface extraction (e.g., [Marching Cubes](53-marching-cubes.md)) by preserving internal volume information [S1], and from splatting ([Splatting](10-splatting.md)) by avoiding sorting and blending overheads for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G].

## How it works
The method consists of two distinct implementation patterns found in the sources: a single-pass fragment shader approach for direct visualization [S2, unofficial] and a multi-pass light propagation approach for global illumination [S1].

### Inputs and Outputs
**Inputs:**
*   **Volume Data:** A 3D texture (e.g., `volume` in texture unit 0) containing scalar density/intensity values [S2, unofficial].
*   **Jitter Data:** A 2D texture (e.g., `jitter` in texture unit 1) containing random values for anti-aliasing [S2, unofficial].
*   **Camera Parameters:** View matrix, Model-View-Projection (MVP) matrix, aspect ratio, focal length, viewport size, ray origin, and bounding box coordinates (top/bottom) [S2, unofficial].
*   **Rendering Parameters:** Step length, threshold, gamma, background color, material color, and light position [S2, unofficial].

**Outputs:**
*   **Final Image:** A color buffer with alpha channel [S2, unofficial].
*   **Light Volumes (Multi-pass only):** Intermediate 3D textures storing local piecewise opacity ($\alpha_{lp}$), global opacity ($\alpha_g$), and final intensity ($I_{final}$) [S1].

### Data Structures
*   **Ray:** Defined by origin ($\mathbf{o}$) and direction ($\mathbf{d}$) [S2, unofficial].
*   **AABB (Axis-Aligned Bounding Box):** Defined by `top` and `bottom` vectors representing the volume bounds [S2, unofficial].
*   **Color/Alpha Accumulator:** A `vec4` storing accumulated color (RGB) and opacity (A) [S2, unofficial].

### Numbered Steps

#### Pattern A: Single-Pass Ray Marching [S2, unofficial]
1.  **Initialize OpenGL State:** Create volume data structure and generate noise if required [S2, unofficial].
2.  **Compute Matrices:**
    *   Compute View Matrix: Translate along Z-axis by $-4.0 \times \exp(\text{distExp} / 600.0)$ [S2, unofficial].
    *   Compute MVP Matrix: Perspective projection with FOV (`m_fov`), aspect ratio (`scaled_width()/scaled_height()`), near plane 0.1, far plane 100.0 [S2, unofficial].
3.  **Compute Ray Origin:** Invert the View Matrix and transform the world origin $(0,0,0)$ to get the ray origin in world space [S2, unofficial].
4.  **Bind Shaders and Uniforms:** Bind the selected shader program (Isosurface, Alpha blending, or MIP) and set uniforms: `ViewMatrix`, `ModelViewProjectionMatrix`, `NormalMatrix`, `aspect_ratio`, `focal_length`, `viewport_size`, `ray_origin`, `top`, `bottom`, `background_colour`, `light_position`, `material_colour`, `step_length`, `threshold`, `gamma`, `volume`, `jitter` [S2, unofficial].
5.  **Clear Buffer:** Clear the color buffer with the background color [S2, unofficial].
6.  **Execute Raycasting (Fragment Shader):**
    *   **Ray Direction:** Compute direction from fragment coordinates:
        $$ \mathbf{d}_{xy} = 2.0 \frac{\mathbf{gl\_FragCoord}_{xy}}{\text{viewport\_size}} - 1.0 $$
        $$ d_x = d_{xy,x} \times \text{aspect\_ratio} $$
        $$ d_z = -\text{focal\_length} $$
        Transform $\mathbf{d}$ by the View Matrix to get world-space direction [S2, unofficial].
    *   **Ray-Box Intersection:** Use slab method to find entry ($t_0$) and exit ($t_1$) distances [S2, unofficial]:
        $$ \mathbf{d}_{inv} = 1.0 / \mathbf{d} $$
        $$ \mathbf{t}_{top} = \mathbf{d}_{inv} \times (\text{box.top} - \mathbf{o}) $$
        $$ \mathbf{t}_{bottom} = \mathbf{d}_{inv} \times (\text{box.bottom} - \mathbf{o}) $$
        $$ t_0 = \max(0.0, \max(\min(\mathbf{t}_{top}, \mathbf{t}_{bottom}).x, \min(\mathbf{t}_{top}, \mathbf{t}_{bottom}).y)) $$
        $$ t_1 = \min(\max(\mathbf{t}_{top}, \mathbf{t}_{bottom}).x, \max(\mathbf{t}_{top}, \mathbf{t}_{bottom}).y) $$
    *   **Normalize to Texture Space:**
        $$ \mathbf{ray}_{start} = (\mathbf{o} + \mathbf{d} t_0 - \text{bottom}) / (\text{top} - \text{bottom}) $$
        $$ \mathbf{ray}_{stop} = (\mathbf{o} + \mathbf{d} t_1 - \text{bottom}) / (\text{top} - \text{bottom}) $$
        $$ \mathbf{ray}_{vec} = \mathbf{ray}_{stop} - \mathbf{ray}_{start} $$
        $$ \text{ray\_length} = \|\mathbf{ray}_{vec}\| $$
        $$ \mathbf{step\_vec} = \text{step\_length} \times \mathbf{ray}_{vec} / \text{ray\_length} $$
    *   **Jitter:** Add random offset:
        $$ \mathbf{ray}_{start} += \mathbf{step\_vec} \times \text{jitter}(\mathbf{gl\_FragCoord}_{xy} / \text{viewport\_size}).r $$
    *   **Marching Loop:** While `ray_length > 0` and `accumulated_alpha < 1.0`:
        *   Sample volume: $I = \text{texture}(\text{volume}, \mathbf{position}).r$ [S2, unofficial].
        *   Transfer Function: Compute color $\mathbf{c}$ and opacity $\alpha_c$ from $I$ [S2, unofficial].
        *   Composite (Front-to-Back):
            $$ \mathbf{C}_{out} = \alpha_c \mathbf{c} + (1 - \alpha_c) \mathbf{C}_{in} $$
            $$ \alpha_{out} = \alpha_c + (1 - \alpha_c) \alpha_{in} $$
        *   Update position: $\mathbf{position} += \mathbf{step\_vec}$, `ray_length -= step_length` [S2, unofficial].
    *   **Background Blend:**
        $$ \mathbf{C}_{final} = \alpha_{out} \mathbf{C}_{out} + (1 - \alpha_{out}) \text{pow}(\text{background\_colour}, \gamma) $$
        $$ \mathbf{C}_{final} = \text{pow}(\mathbf{C}_{final}, 1/\gamma) $$ [S2, unofficial].

#### Pattern B: Multi-Pass Light Propagation [S1]
1.  **Light Update Order:** Compute light source direction first (for shadows), then incrementally add scattering directions by subdividing a tetrahedron/icosahedron/octahedron [S1].
2.  **Execute Shader Programs (One per slice):**
    *   **Step 1:** Compute local piecewise opacity $\alpha_{lp}$ (Eq. 6) [S1].
    *   **Step 2:** Compute global opacity $\alpha_g$ (Eq. 7) [S1].
    *   **Step 3:** Compute final intensity $I_{final}$ (Eq. 12) [S1].
    *   *Note:* Each slice is rendered one-by-one to a Frame Buffer Object (FBO) attached to a 3D texture slice [S1].
3.  **Incremental Refinement:** Blend new $I_{final}$ results with stored intensities $I_{stored}$ using Eq. 13 to update $I_{blended}$ [S1].
4.  **Final Ray Casting:** Execute a final shader program to compute $I_{eye}$ using the stored light volumes [S1].

### Formulas
*   **Ray-Box Intersection (Slab Method):**
    $$ t_0 = \max(0.0, \max(\min(\mathbf{d}_{inv}(\mathbf{b}_{top}-\mathbf{o}), \mathbf{d}_{inv}(\mathbf{b}_{bottom}-\mathbf{o})).x, \dots)) $$
    $$ t_1 = \min(\max(\mathbf{d}_{inv}(\mathbf{b}_{top}-\mathbf{o}), \mathbf{d}_{inv}(\mathbf{b}_{bottom}-\mathbf{o})).x, \dots) $$ [S2, unofficial].
*   **Transfer Function (Simple):**
    $$ \alpha = \frac{\exp(I) - 1.0}{\exp(1.0) - 1.0} $$
    $$ \mathbf{c} = I \cdot \mathbf{high} + (1.0 - I) \cdot \mathbf{low} $$ [S2, unofficial].
*   **Light Blending (Incremental):**
    $$ I_{blended} = \dots \text{(Eq. 13 in [S1])} $$ [S1].

### Parameters
*   **Step Length:** Controls sampling density. Typical values: 8-16 voxels for piecewise segments [S1].
*   **Scattering Sphere Radius ($R_\Omega$):** Adjusts shadow distinctness [S1].
*   **Bias ($I_{bias}$):** Adjusts minimum light level [S1].
*   **Data Reduction:** 8.9:1 reduction (e.g., 512^3 to 256x256x128) is feasible without significant visual loss [S1].

### Edge Cases and Pitfalls
*   **Empty Blocks:** In multi-resolution structures, rays originating in empty blocks may miss occlusion in neighboring non-empty blocks. Solution: Advance ray to next non-empty block [S1].
*   **Interpolation Artifacts:** Flat blocking multi-resolution management can cause interpolation problems at block boundaries [S1].
*   **Precision:** Linear interpolation requires a regular linear volume to avoid artifacts [S1].

## Subtopics

### [Volume Rendering with 3D Textures and Ray Marching](2584-volume-rendering-with-3d-textures-and-ray-marching.md)
Performs object-order rendering by ray marching through a 3D texture, applying a transfer function and compositing front-to-back. Solves visualization of volumetric data without mesh generation. Preserves internal volume info unlike isosurface extraction. Avoids splatting sorting overheads but is heavier per pixel.

### [NVIDIA GeForce3 Volume Rendering Pipeline](5630-nvidia-geforce3-volume-rendering-pipeline.md)
Hardware-accelerated method using per-fragment dependent texture reads and 4-way multi-texturing. Eliminates scratch buffers by computing texture lookups dynamically. Trades flexibility of arbitrary 3D transfer functions for performance, constraining them to separable 2D and 1D components.

### [Real-Time Rendering Pipeline with Virtual Memory](8407-real-time-rendering-pipeline-with-virtual-memory.md)
Implements real-time rendering for 3D Gaussian Splatting by streaming data via virtual memory. Maps logical page IDs to physical GPU memory, using LRU replacement. Performs global distance sorting in a compute shader before rasterization, replacing software tile-sorting.

## Operating characteristics
*   **Time Complexity:** Depends on volume size, step length, and data reduction. For a 512^3 volume with 8.9:1 reduction and SVR 256^3: Global light integration takes 284-552 ms; incremental refinements take 62-233 ms [S1].
*   **Memory Complexity:** Storage can be reduced via multi-resolution structures. A 512^3 dataset can be stored as 256x256x128 voxels without significant loss [S1].
*   **Performance:** High frame rates (15-22 fps) reached after light updates on Nvidia GeForce 8800 Ultra with 768 MB graphics texture memory [S1].
*   **Strengths:** Single-pass efficiency for direct visualization [S2, unofficial]; hardware acceleration via dependent texture reads [S2, unofficial]; virtual memory support for large scenes [S1].
*   **Limitations:** Computationally heavier per pixel than splatting [G]; interpolation artifacts at multi-resolution block boundaries [S1]; constraints on transfer function complexity due to hardware limits [S2, unofficial].
*   **When to Use:** Interactive visualization of dense volumetric data (medical, scientific) where internal structure is important [S4].
*   **When Not to Use:** When extreme performance is required for sparse data (use splatting) or when isosurfaces are sufficient [S1].

## Background
The method originates from GPU-accelerated volumetric raycasting techniques implemented in GLSL and C++ [S2, unofficial]. It builds on the principle of stage separation, utilizing fragment shaders for direct computation [S1, S3, unofficial]. The multi-pass light propagation variant was introduced to handle global illumination in direct volume rendering using local piecewise integration [S1].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S2] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
