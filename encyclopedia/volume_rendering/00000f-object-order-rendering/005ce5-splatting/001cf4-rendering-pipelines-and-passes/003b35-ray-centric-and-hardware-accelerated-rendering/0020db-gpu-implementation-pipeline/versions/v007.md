# GPU Implementation Pipeline

Also called: Single-pass GPU Raycasting, Volumetric Raycasting

## Overview
A GPU-accelerated rendering technique that traces rays from the camera through each pixel into a 3D volume to compute color and opacity, implemented in a single pass using fragment shaders [S2, unofficial]. It solves the problem of real-time visualization of volumetric data by leveraging hardware rasterization for ray-box intersection and texture sampling, avoiding the overhead of intermediate scratch buffers or complex sorting required by splatting methods [S1][S2, unofficial]. Unlike [Batched Ray Rendering](1547-batched-ray-rendering.md), which may group rays for optimization, this pipeline typically processes pixels or small batches in a direct fragment shader execution model [S2, unofficial].

## How it works
The method consists of two primary implementation patterns found in the sources: a direct single-pass raycaster [S2, unofficial] and a multi-pass light propagation pipeline [S1]. The direct single-pass approach is the standard "single-pass GPU raycasting" [S2, unofficial].

### Inputs and Outputs
**Inputs:**
- Volume data: 3D texture (e.g., `GL_TEXTURE_3D`) storing scalar density/intensity values [S2, unofficial].
- Camera parameters: View matrix, Model-View-Projection (MVP) matrix, aspect ratio, focal length, viewport size [S2, unofficial].
- Rendering parameters: Step length, near/far planes, background color, gamma, transfer function coefficients [S2, unofficial].
- Optional: Jitter texture for anti-aliasing [S2, unofficial].

**Outputs:**
- Final image: Color buffer with computed RGB and alpha values [S2, unofficial].
- Intermediate light volumes (in light propagation variant): $\alpha_{lp}$, $\alpha_g$, $I_{final}$ stored in Frame Buffer Objects (FBOs) attached to 3D texture slices [S1].

### Data Structures
1.  **Volume Texture:** 3D texture object. In the light propagation method, a multi-resolution structure is used where a $512^3$ dataset can be stored as $256 \times 256 \times 128$ voxels for 8.9:1 data reduction [S1].
2.  **Ray Structure:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [S2, unofficial].
3.  **AABB (Axis-Aligned Bounding Box):** Defined by `top` and `bottom` vectors representing the volume bounds in world space [S2, unofficial].
4.  **Frame Buffer Object (FBO):** Used in the light propagation method to render slices to 3D texture targets [S1].

### Numbered Steps (Direct Single-Pass Raycasting)
1.  **Initialize OpenGL State:** Create volume data structure and generate noise if needed [S2, unofficial].
2.  **Compute Matrices:**
    -   Compute View Matrix: Translate along Z-axis based on distance exponent $m_{distExp}$:
        $$ \text{translate}(0, 0, -4.0 \cdot \exp(m_{distExp} / 600.0)) $$
        [S2, unofficial].
    -   Compute MVP Matrix: Perspective projection with field of view $m_{fov}$, aspect ratio $\text{scaled\_width}/\text{scaled\_height}$, near plane $0.1$, far plane $100.0$ [S2, unofficial].
3.  **Compute Ray Origin:** Invert the View Matrix and transform the world origin $(0,0,0)$ to get the ray origin in world space [S2, unofficial].
4.  **Bind Shader and Set Uniforms:** Bind the fragment shader program. Set uniforms: `ViewMatrix`, `ModelViewProjectionMatrix`, `NormalMatrix`, `aspect_ratio`, `focal_length`, `viewport_size`, `ray_origin`, `top`, `bottom`, `background_colour`, `light_position`, `material_colour`, `step_length`, `threshold`, `gamma`, `volume` (texture unit 0), `jitter` (texture unit 1) [S2, unofficial].
5.  **Clear Buffer:** Clear the color buffer with `background_colour` [S2, unofficial].
6.  **Execute Raycasting Shader (Per Pixel):**
    -   **Ray Direction:** Compute from fragment coordinates $(x,y)$:
        $$ \mathbf{d}_{xy} = 2.0 \cdot \frac{(x,y)}{\text{viewport\_size}} - 1.0 $$
        $$ \mathbf{d}_x = \mathbf{d}_{xy}.x \cdot \text{aspect\_ratio} $$
        $$ \mathbf{d}_z = -\text{focal\_length} $$
        Transform $\mathbf{d}$ by `ViewMatrix` and normalize [S2, unofficial].
    -   **Ray-Box Intersection:** Use slab method to find entry $t_0$ and exit $t_1$ times for the AABB [S2, unofficial]:
        $$ \mathbf{d}_{inv} = 1.0 / \mathbf{d} $$
        $$ \mathbf{t}_{top} = \mathbf{d}_{inv} \cdot (\text{box.top} - \mathbf{o}) $$
        $$ \mathbf{t}_{bottom} = \mathbf{d}_{inv} \cdot (\text{box.bottom} - \mathbf{o}) $$
        $$ \mathbf{t}_{min} = \min(\mathbf{t}_{top}, \mathbf{t}_{bottom}) $$
        $$ t_0 = \max(0.0, \max(\mathbf{t}_{min}.x, \mathbf{t}_{min}.y)) $$
        $$ \mathbf{t}_{max} = \max(\mathbf{t}_{top}, \mathbf{t}_{bottom}) $$
        $$ t_1 = \min(\mathbf{t}_{max}.x, \mathbf{t}_{max}.y) $$
        [S2, unofficial].
    -   **Ray Parameters:** Calculate start/stop positions in texture coordinates and step vector [S2, unofficial]:
        $$ \mathbf{r}_{start} = \frac{\mathbf{o} + \mathbf{d} \cdot t_0 - \text{bottom}}{\text{top} - \text{bottom}} $$
        $$ \mathbf{r}_{stop} = \frac{\mathbf{o} + \mathbf{d} \cdot t_1 - \text{bottom}}{\text{top} - \text{bottom}} $$
        $$ \mathbf{r} = \mathbf{r}_{stop} - \mathbf{r}_{start} $$
        $$ L = \|\mathbf{r}\| $$
        $$ \mathbf{s} = \text{step\_length} \cdot \frac{\mathbf{r}}{L} $$
        [S2, unofficial].
    -   **Jitter:** Add random offset from jitter texture to $\mathbf{r}_{start}$ [S2, unofficial].
    -   **Ray Marching Loop:**
        Initialize `colour` = $(0,0,0,0)$ [S2, unofficial].
        While $L > 0$ and `colour.a` < $1.0$:
        1.  Sample volume: $I = \text{texture}(\text{volume}, \mathbf{r}_{start}).r$ [S2, unofficial].
        2.  Transfer Function: Compute color $C$ and opacity $\alpha$ from $I$. Example:
            $$ \alpha = \frac{\exp(I) - 1.0}{\exp(1.0) - 1.0} $$
            $$ C = I \cdot \text{high} + (1.0 - I) \cdot \text{low} $$
            [S2, unofficial].
        3.  Alpha Blending (Front-to-Back):
            $$ \text{colour.rgb} = \alpha \cdot C + (1 - \alpha) \cdot \text{colour.a} \cdot \text{colour.rgb} $$
            $$ \text{colour.a} = \alpha + (1 - \alpha) \cdot \text{colour.a} $$
            [S2, unofficial].
        4.  Update: $L \leftarrow L - \text{step\_length}$, $\mathbf{r}_{start} \leftarrow \mathbf{r}_{start} + \mathbf{s}$ [S2, unofficial].
    -   **Final Blend:** Blend with background and apply gamma [S2, unofficial]:
        $$ \text{colour.rgb} = \text{colour.a} \cdot \text{colour.rgb} + (1 - \text{colour.a}) \cdot \text{pow}(\text{background\_colour}, \gamma).\text{rgb} $$
        $$ \text{colour.a} = 1.0 $$
        $$ \text{output.rgb} = \text{pow}(\text{colour.rgb}, 1/\gamma) $$
        [S2, unofficial].

### Numbered Steps (Light Propagation Variant)
1.  **Light Update Order:** Compute light source direction first for shadows [S1].
2.  **Incremental Scattering:** Subdivide a tetrahedron/icosahedron/octahedron to add scattering directions [S1].
3.  **Shader Execution (3 Passes):**
    -   Pass 1: Compute local piecewise opacity $\alpha_{lp}$ [S1].
    -   Pass 2: Compute global opacity $\alpha_g$ [S1].
    -   Pass 3: Compute final intensity $I_{final}$ including scattering [S1].
    -   Each slice is rendered to an FBO attached to a 3D texture slice [S1].
4.  **Final Ray Casting:** Execute shader to compute $I_{eye}$ using stored light volumes [S1].
5.  **Incremental Update:** Blend new $I_{final}$ with stored $I_{stored}$ using:
    $$ I_{blended} = \dots $$
    (Formula not fully provided in source, but blending is required) [S1].

### Parameters and Typical Values
-   **Step Length:** Determines sampling density. Smaller values increase accuracy but decrease performance [S2, unofficial].
-   **Data Reduction:** 8.9:1 reduction (e.g., $512^3$ to $256 \times 256 \times 128$) is feasible without significant visual loss [S1].
-   **Piecewise Segment Length:** 8-16 voxels provides good performance/quality trade-off [S1].
-   **Hardware:** Tested on Nvidia GeForce 8800 Ultra with 768 MB graphics texture memory [S1].
-   **Performance:** For $512^3$ volume, Global light integration (A) takes 284-552 ms. First order scattering refinements (B) take 62-233 ms. High frame rates (15-22 fps) reached after light update [S1].

### Edge Cases and Pitfalls
-   **Empty Blocks:** In multi-resolution structures, rays originating in empty blocks may cover neighboring non-empty blocks, losing occlusion. Solution: Advance ray to next non-empty block [S1].
-   **Interpolation Artifacts:** May occur with flat blocking multi-resolution data management [S1].
-   **Scattering Sphere Radius:** Large radius makes shadows less distinct [S1].
-   **Bias:** $I_{bias}$ adjusts minimum intensity [S1].

## Subtopics

### [Volume Rendering with 3D Textures and Ray Marching](2584-volume-rendering-with-3d-textures-and-ray-marching.md)
Performs object-order rendering by ray marching through a 3D texture, applying transfer functions and front-to-back compositing. Solves visualization without polygonal meshes. Differs from isosurface extraction by preserving internal volume and from splatting by avoiding sorting overhead, though heavier per pixel.

### [NVIDIA GeForce3 Volume Rendering Pipeline](5630-nvidia-geforce3-volume-rendering-pipeline.md)
Hardware-accelerated method using per-fragment dependent texture reads and 4-way multi-texturing to eliminate intermediate scratch buffers. Distinguishes itself by dynamic texture lookups based on fragment attributes, avoiding expensive memory copies.

### [Real-Time Rendering Pipeline with Virtual Memory](8407-real-time-rendering-pipeline-with-virtual-memory.md)
Implements real-time rendering for 3D Gaussian Splatting by streaming data via virtual memory. Solves memory bottlenecks by mapping logical page IDs to physical GPU memory, using LRU replacement. Performs global distance sorting in a compute shader before rasterization.

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of samples per ray and number of pixels. Performance depends on step length and volume resolution [S2, unofficial].
-   **Memory Complexity:** Proportional to volume size. Multi-resolution structures can reduce storage by factors of 8 or more [S1].
-   **Measured Performance:** On Nvidia GeForce 8800 Ultra, $512^3$ volume with 8.9:1 reduction achieves 15-22 fps after light update initialization [S1].
-   **Strengths:** Real-time interactivity, single-pass efficiency for direct raycasting, no sorting required [S2, unofficial].
-   **Limitations:** Performance degrades with high resolution or small step sizes. Multi-resolution approaches may introduce interpolation artifacts or occlusion errors in empty blocks [S1].
-   **When to Use:** Real-time visualization of dense volumes where sorting overhead is prohibitive [S2, unofficial].
-   **When Not to Use:** When extreme visual fidelity with complex lighting (global illumination) is required without pre-computation, or when memory bandwidth is severely constrained without multi-resolution support [S1].

## Background
The method originates from GPU-accelerated volume rendering techniques that leverage fragment shaders for direct ray casting [S2, unofficial]. The single-pass approach is a common implementation pattern for real-time volume visualization [S2, unofficial]. The light propagation variant was introduced to handle global illumination effects efficiently using local piecewise integration and multi-resolution data structures [S1].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S2] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
