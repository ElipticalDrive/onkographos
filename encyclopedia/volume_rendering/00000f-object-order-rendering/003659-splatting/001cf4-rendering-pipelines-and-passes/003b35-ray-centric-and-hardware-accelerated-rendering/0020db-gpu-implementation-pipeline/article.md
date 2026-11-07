# GPU Implementation Pipeline

Also called: Single-pass GPU Raycasting, Volumetric Raycasting

## Overview
A GPU-accelerated rendering technique that traces rays from the camera through each pixel into a 3D volume to compute color and opacity, implemented in a single pass using fragment shaders [S2, unofficial]. It solves the problem of visualizing volumetric data directly on the GPU without generating intermediate polygonal meshes [S4, unofficial]. It differs from isosurface extraction (e.g., [Marching Cubes](53-marching-cubes.md)) by preserving internal volume information [S1], and from splatting ([Splatting](10-splatting.md)) by avoiding sorting and blending overheads for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G]. It builds on the [Ray-Centric and Hardware-Accelerated Rendering](8936-ray-centric-and-hardware-accelerated-rendering.md) family principles.

## How it works
**Inputs:**
- 3D Volume Texture: A 3D texture containing scalar density/intensity values (e.g., medical DICOM series or synthetic fields) [S2, unofficial].
- Camera Parameters: View matrix, projection matrix, viewport size, aspect ratio, focal length, and background color [S2, unofficial].
- Rendering Parameters: Step length, threshold for opacity termination, gamma correction value, and transfer function coefficients [S2, unofficial].

**Outputs:**
- Final Image: A 2D color buffer with the composited volume rendering result [S2, unofficial].

**Data Structures:**
- `Ray`: Struct containing `origin` (vec3) and `direction` (vec3) [S2, unofficial].
- `AABB`: Axis-Aligned Bounding Box defined by `top` (vec3) and `bottom` (vec3) corners [S2, unofficial].
- `Jitter Texture`: A 2D texture storing random values for jittering ray origins to reduce aliasing [S2, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    -   Compute the view matrix by translating along the Z-axis based on an exponential distance exponent: $T_z = -4.0 \cdot \exp(\text{distExp} / 600.0)$ [S2, unofficial].
    -   Compute the model-view-projection matrix using a perspective projection with field of view (FOV), aspect ratio ($\text{scaled\_width} / \text{scaled\_height}$), near plane $0.1$, and far plane $100.0$ [S2, unofficial].
    -   Compute the ray origin in world space by inverting the view matrix and transforming the origin $(0,0,0)$ [S2, unofficial].
    -   Bind the shader program and set uniform values: `ViewMatrix`, `ModelViewProjectionMatrix`, `NormalMatrix`, `aspect_ratio`, `focal_length`, `viewport_size`, `ray_origin`, `top`, `bottom`, `background_colour`, `step_length`, `threshold`, `gamma`, `volume` (texture unit 0), and `jitter` (texture unit 1) [S2, unofficial].
    -   Clear the color buffer with the background color [S2, unofficial].

2.  **Ray Generation (Per-Pixel):**
    -   For each fragment, compute the ray direction in screen space:
        $$
        \begin{aligned}
        x &= 2.0 \cdot \frac{\text{gl\_FragCoord}.x}{\text{viewport\_size}.x} - 1.0 \\
        y &= 2.0 \cdot \frac{\text{gl\_FragCoord}.y}{\text{viewport\_size}.y} - 1.0
        \end{aligned}
        $$
    -   Adjust for aspect ratio and focal length:
        $$
        \begin{aligned}
        \text{ray\_direction}.xy &= (x \cdot \text{aspect\_ratio}, y) \\
        \text{ray\_direction}.z &= -\text{focal\_length}
        \end{aligned}
        $$
    -   Transform the direction to world space by multiplying with the `ViewMatrix` (treating direction as a vector with $w=0$) [S2, unofficial].

3.  **Bounding Box Intersection:**
    -   Test the ray against the AABB enclosing the volume using the slab method [S2, unofficial].
    -   Compute reciprocal direction: $\text{direction\_inv} = 1.0 / \text{ray.direction}$ [S2, unofficial].
    -   Compute intersection distances with box planes:
        $$
        \begin{aligned}
        t_{\text{top}} &= \text{direction\_inv} \cdot (\text{box.top} - \text{ray.origin}) \\
        t_{\text{bottom}} &= \text{direction\_inv} \cdot (\text{box.bottom} - \text{ray.origin})
        \end{aligned}
        $$
    -   Determine entry ($t_0$) and exit ($t_1$) distances:
        $$
        \begin{aligned}
        t_{\min} &= \min(t_{\text{top}}, t_{\text{bottom}}) \\
        t_{\max} &= \max(t_{\text{top}}, t_{\text{bottom}}) \\
        t_0 &= \max(0.0, \max(t_{\min}.x, t_{\min}.y)) \\
        t_1 &= \min(t_{\max}.x, t_{\max}.y)
        \end{aligned}
        $$
    -   If $t_0 > t_1$, the ray misses the volume [S2, unofficial].

4.  **Ray Marching Setup:**
    -   Calculate the start and end positions of the ray within the volume's normalized texture coordinate space $[0,1]^3$:
        $$
        \begin{aligned}
        \text{ray\_start} &= \frac{\text{ray.origin} + \text{ray.direction} \cdot t_0 - \text{bottom}}{\text{top} - \text{bottom}} \\
        \text{ray\_stop} &= \frac{\text{ray.origin} + \text{ray.direction} \cdot t_1 - \text{bottom}}{\text{top} - \text{bottom}}
        \end{aligned}
        $$
    -   Compute the ray vector within texture space: $\text{ray} = \text{ray\_stop} - \text{ray\_start}$ [S2, unofficial].
    -   Compute the total length in texture space units: $\text{ray\_length} = \|\text{ray}\|$ [S2, unofficial].
    -   Compute the step vector in texture space:
        $$
        \text{step\_vector} = \text{step\_length} \cdot \frac{\text{ray}}{\text{ray\_length}}
        $$
    -   Apply random jitter to the start position using the `jitter` texture to reduce aliasing:
        $$
        \text{position} = \text{ray\_start} + \text{step\_vector} \cdot \text{jitter}(\frac{\text{gl\_FragCoord}}{\text{viewport\_size}}).r
        $$
        [S2, unofficial].

5.  **Front-to-Back Compositing Loop:**
    -   Initialize accumulated color $C_{acc} = (0,0,0,0)$ [S2, unofficial].
    -   While $\text{ray\_length} > 0$ and $C_{acc}.a < 1.0$:
        1.  Sample the volume texture at `position`: $I = \text{texture}(\text{volume}, \text{position}).r$ [S2, unofficial].
        2.  Apply the transfer function to get color $C$ and opacity $\alpha$. A simple exponential transfer function is:
            $$
            \begin{aligned}
            \alpha &= \frac{\exp(I) - 1.0}{\exp(1.0) - 1.0} \\
            C &= I \cdot \text{high} + (1.0 - I) \cdot \text{low}
            \end{aligned}
            $$
            where $\text{high}=(1,1,1)$ and $\text{low}=(0,0,0)$ [S2, unofficial].
        3.  Update accumulated color and opacity using alpha blending:
            $$
            \begin{aligned}
            C_{acc}.rgb &= C.a \cdot C.rgb + (1 - C.a) \cdot C_{acc}.a \cdot C_{acc}.rgb \\
            C_{acc}.a &= C.a + (1 - C.a) \cdot C_{acc}.a
            \end{aligned}
            $$
            [S2, unofficial].
        4.  Advance the ray:
            $$
            \begin{aligned}
            \text{ray\_length} &\leftarrow \text{ray\_length} - \text{step\_length} \\
            \text{position} &\leftarrow \text{position} + \text{step\_vector}
            \end{aligned}
            $$
            [S2, unofficial].

6.  **Final Compositing and Output:**
    -   Blend the accumulated volume color with the background color:
        $$
        \begin{aligned}
        \text{final\_rgb} &= C_{acc}.a \cdot C_{acc}.rgb + (1 - C_{acc}.a) \cdot \text{pow}(\text{background\_colour}, \text{gamma}).rgb \\
        \text{final\_a} &= 1.0
        \end{aligned}
        $$
        [S2, unofficial].
    -   Apply gamma correction to the final color:
        $$
        \text{output\_rgb} = \text{pow}(\text{final\_rgb}, \frac{1.0}{\text{gamma}})
        $$
        [S2, unofficial].

**Parameters with Typical Values:**
-   `step_length`: Determines sampling density. Smaller values increase quality but reduce performance. Not given in the sources.
-   `threshold`: Opacity threshold for early termination. Typically $1.0$ [S2, unofficial].
-   `gamma`: Gamma correction value. Typically $2.2$ [S2, unofficial].
-   `distExp`: Exponential distance exponent for camera translation. Not given in the sources.
-   `m_fov`: Field of view. Not given in the sources.

**Edge Cases and Pitfalls:**
-   **Zero Direction Components:** The slab method uses reciprocal direction ($1.0 / \text{ray.direction}$). If a direction component is zero, this results in infinity. The min/max logic in the slab method handles this correctly if implemented as shown, but care must be taken with floating-point precision [S2, unofficial].
-   **Empty Blocks:** In multi-resolution structures, rays may originate in empty blocks. The ray must be advanced to the next non-empty block to avoid artifacts from lost occlusion [S1].
-   **Precision:** Linear interpolation in multi-resolution structures can cause artifacts if blocks are not handled correctly [S1].

## Subtopics

### [Volume Rendering with 3D Textures and Ray Marching](2584-volume-rendering-with-3d-textures-and-ray-marching.md)
This method performs object-order volume rendering by ray marching through a 3D texture stored in GPU memory, applying a transfer function to map scalar values to color and opacity, and compositing the result front-to-back. It solves the problem of visualizing volumetric data (such as medical DICOM series or synthetic density fields) directly on the GPU without generating intermediate polygonal meshes. It differs from isosurface extraction (e.g., [Marching Cubes](53-marching-cubes.md)) by preserving internal volume information, and from splatting ([Splatting](10-splatting.md)) by avoiding sorting and blending overheads for dense volumes, though it is computationally heavier per pixel than texture-based splatting [G].

### [NVIDIA GeForce3 Volume Rendering Pipeline](5630-nvidia-geforce3-volume-rendering-pipeline.md)
The NVIDIA GeForce3 Volume Rendering Pipeline is a hardware-accelerated method for interactive volume rendering that utilizes per-fragment dependent texture reads and 4-way multi-texturing to eliminate the need for intermediate scratch buffers. It distinguishes itself from prior software-based or scratch-buffer-dependent approaches (such as the SGI Octane2 method) by performing texture lookups dynamically based on computed fragment attributes, thereby avoiding expensive memory copy operations. This approach trades the flexibility of arbitrary 3D transfer functions for performance gains, constraining the transfer function to a separable product of 2D and 1D components due to hardware limitations on dependent texture dimensions.

### [Real-Time Rendering Pipeline with Virtual Memory](8407-real-time-rendering-pipeline-with-virtual-memory.md)
This method implements a real-time rendering pipeline for 3D Gaussian Splatting that streams Gaussian data to GPU memory on demand using a virtual memory system. It solves the memory capacity bottleneck of large scenes by mapping logical page IDs to physical GPU memory pages, replacing unused pages based on a least-recently-used (LRU) policy. Unlike software-based tile-sorting approaches (e.g., Kerbl et al.), this method performs global distance sorting of Gaussians in a GLSL compute shader before splatting them via the hardware rasterizer.

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of samples per ray and the number of pixels. The number of samples is determined by the ray length through the volume and the `step_length` [S2, unofficial].
-   **Memory Complexity:** Dominated by the storage of the 3D volume texture and the jitter texture. The volume texture size depends on the resolution of the input data [S2, unofficial].
-   **Measured Performance:** For a $512^3$ volume with 8.9:1 data reduction, SVR size $256^3$, and piecewise segment length of 8 voxels, global light integration takes 284-552 ms, and first-order scattering incremental refinements take 62-233 ms. High frame rates (15-22 fps) are reached after light updates [S1].
-   **Hardware:** Tested on Nvidia GeForce 8800 Ultra with 768 MB graphics texture memory [S1].
-   **Strengths:** Single-pass implementation allows for efficient per-pixel processing. Hardware acceleration via fragment shaders provides interactive frame rates for moderate resolution volumes [S2, unofficial].
-   **Limitations:** Computationally heavy per pixel compared to splatting for dense volumes. Performance degrades with high resolution volumes or small step lengths. Requires sufficient GPU memory for the 3D texture [S2, unofficial].
-   **When to Use:** Suitable for real-time visualization of volumetric data where internal structure is important and the volume fits in GPU memory.
-   **When Not to Use:** Not suitable for extremely large volumes that exceed GPU memory capacity without multi-resolution streaming, or for scenes requiring complex light transport beyond simple absorption and emission [S1].

## Background
The GPU-accelerated single-pass raycasting technique is a modern implementation of volumetric raycasting, leveraging the parallel processing capabilities of GPUs [S2, unofficial]. It builds upon earlier software-based raycasting methods and hardware-accelerated pipelines like the NVIDIA GeForce3 Volume Rendering Pipeline [S1][S2, unofficial]. The method is commonly used in medical imaging visualization and scientific data analysis [S4, unofficial]. The specific implementation described here is based on a simple visualizer project [S2, unofficial].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S2] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
