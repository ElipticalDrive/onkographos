# Planes Method

Also called: Compositing on the CPU, RealityEngine Volume Rendering with CPU Compositing

## Overview
The Planes Method is a hardware-accelerated volume rendering technique that leverages 3D texture mapping hardware to resample volumetric data into planes coplanar with rays ending in a row of pixels, then performs shading and front-to-back compositing on the CPU [S1]. It solves the problem of view-dependent shading and interactive classification in texture-mapped rendering by offloading the computationally intensive Phong shading and compositing logic to the processor, allowing for optimization techniques like early ray termination and low-opacity skipping [S1]. Unlike [Shear-Warp](shear-warp.md), which performs classification before bilinear resampling and requires a final warp step, the Planes method performs trilinear interpolation followed by classification and requires no final warp, resulting in higher image quality and consistent performance for perspective projections [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volumetric dataset where each voxel consists of four 8-bit values: three gradient components ($G_x, G_y, G_z$) and raw density ($\rho$) [S1].
*   **Input:** A viewing volume defined by parallel or perspective projection parameters [S1].
*   **Input:** Transfer functions for opacity and reflectance (Phong shading parameters) [S1].
*   **Output:** A 2D image frame buffer with composited color and opacity values [S1].

**Data Structures:**
The voxel data is loaded into an RGB-type 3D texture map. Although the texture format is RGB, the fields store three gradient values and raw density [S1]. The gradients are precomputed offline using high-quality Sobel operators [S1].

**Algorithm Steps:**
The method utilizes a master-slave parallel architecture on symmetric multi-processor (SMP) systems, where the master handles texture mapping and slaves handle raycasting [S1]. The core rendering loop processes scanlines sequentially but interlaces CPU and GPU work [S1].

1.  **Preprocessing:** Compute Sobel gradients for the volume offline. Load the gradient/density data into the 3D texture map [S1].
2.  **Initialization:** Load the texture rotation matrix corresponding to the current view [S1].
3.  **First Plane Resampling:** Resample the first plane of orthogonal rays into the frame buffer using 3D texture mapping hardware [S1].
4.  **Memory Transfer:** Read the first plane from the frame buffer into main memory [S1].
5.  **Scanline Loop:** For each remaining scanline $y$:
    a.  **Resample:** Resample the next plane of orthogonal rays into the frame buffer [S1].
    b.  **Column Loop:** For each column $x$ in the previous plane (which corresponds to a ray):
        i.  **Initialize:** Set integration variables (accumulated color $C_{acc}=0$, accumulated opacity $A_{acc}=0$) [S1].
        ii. **Ray Marching Loop:** While the sample position is within bounds and the ray is not opaque ($A_{acc} < 1.0$):
            1.  **Sample:** Retrieve density $\rho$ and gradients $\mathbf{G}$ from the resampled plane data at the current sample position. The hardware performs trilinear interpolation of these values [S1].
            2.  **Opacity Lookup:** Lookup opacity $\alpha$ in the transfer function using $\rho$ [S1].
            3.  **Skip Check:** If $\alpha$ is below a threshold, skip shading and compositing for this sample (space leaping) [S1].
            4.  **Shading:** If $\alpha >$ threshold, lookup shade in reflectance map to generate Phong shaded RGB [S1].
            5.  **Composite:** Update accumulated color and opacity using front-to-back compositing:
                $$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha \cdot (1 - A_{acc}) $$
                $$ A_{acc} = A_{acc} + \alpha \cdot (1 - A_{acc}) $$
            6.  **Early Termination:** If $A_{acc} \ge 1.0$, terminate the ray marching loop (early ray termination) [S1].
        iii. **Store:** Store ray color in previous row of image [S1].
    c.  **Next Plane:** Read next plane from frame buffer into memory [S1].
6.  **Final Plane:** Shade and composite final plane as above [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def planes_method(volume_texture, view_matrix, opacity_func, reflectance_func, image_width, image_height, num_samples, opacity_threshold):
    # Precomputed gradients are in volume_texture as RGB (Gx, Gy, Gz) + Density [S1]
    # Load texture rotation matrix [S1]
    texture_rotation = view_matrix

    # Resample first plane into frame buffer [S1]
    resample_plane_into_framebuffer(0, texture_rotation)

    # Read first plane from frame buffer into memory [S1]
    plane_data = read_framebuffer_to_memory(0)

    # Loop over all remaining scanlines [S1]
    for y in range(1, image_height):
        # Interlace: Initiate texture mapping for next scanline [S1]
        initiate_texture_mapping(y + 1, texture_rotation)

        # Resample next plane into frame buffer [S1]
        resample_plane_into_framebuffer(y, texture_rotation)

        # Loop over all columns in the previous plane [S1]
        for x in range(image_width):
            # Initialize integration variables [S1]
            acc_color = [0.0, 0.0, 0.0]
            acc_opacity = 0.0

            # While within bounds and ray not opaque [S1]
            for s in range(num_samples):
                # Get sample data from current_plane_data at column x, sample s
                density = plane_data[x, s, 3]
                gradients = plane_data[x, s, :3]

                # Lookup opacity in transfer function [S1]
                alpha = opacity_func(density)

                # If sample opacity > threshold: Lookup shade in reflectance map [S1]
                if alpha > opacity_threshold:
                    sample_color = reflectance_func(gradients, view_matrix)
                else:
                    sample_color = [0.0, 0.0, 0.0]

                # Composite ray color OVER sample [S1]
                weight = alpha * (1.0 - acc_opacity)
                acc_color[0] += sample_color[0] * weight
                acc_color[1] += sample_color[1] * weight
                acc_color[2] += sample_color[2] * weight
                acc_opacity += weight

                # Early ray termination (ray not opaque)
                if acc_opacity >= 1.0:
                    break

            # Store ray color in previous row of image [S1]
            set_pixel_color(x, y, acc_color)

        # Read next plane from frame buffer into memory [S1]
        plane_data = read_framebuffer_to_memory(y)

    # Shade and composite final plane as above [S1]

def load_texture_rotation_matrix(matrix):
    # Loads the view matrix into the 3D texture mapping hardware's rotation register [S1]
    pass

def resample_plane_into_framebuffer(scanline_idx, rotation_matrix):
    # Forwards a polygon coplanar with rays ending in a row of pixels to the graphics pipeline.
    # Hardware performs trilinear interpolation of the 3D texture map and writes to frame buffer [S1]
    pass

def read_framebuffer_to_memory(scanline_idx):
    # Reads the resampled plane data from the frame buffer into main memory as a 2D array [width, samples, 4] [S1]
    pass

def initiate_texture_mapping(scanline_idx, rotation_matrix):
    # Queues the texture mapping calculation for the next scanline to interleave CPU and GPU work [S1]
    pass

def set_pixel_color(x, y, color):
    # Writes the accumulated color to the image buffer at pixel coordinates (x, y) [S1]
    pass

def reflectance_map(gradients, view_matrix):
    # Computes Phong shaded RGB using the interpolated gradients and view parameters [S1]
    pass
```

**Parameters:**
*   **Opacity Threshold:** A value below which shading and compositing are skipped [S1].
*   **Gradient Computation:** High-quality Sobel gradients are computed serially offline [S1].
*   **Hardware:** Onyx 2 workstation with Infinite Reality graphics [S1].

**Edge Cases and Pitfalls:**
*   **Texture Memory Limits:** The volume size is limited by available texture memory due to storing gradients, causing texture paging for large volumes [S1].
*   **Sequential Bottleneck:** The parallel speedup is limited by the time it takes to perform texture mapping for all planes, which must be performed sequentially because there is only one graphics pipeline [S1].
*   **Precision:** The method relies on 3D texture mapping hardware for interpolation. The sources state that assuming a piecewise linear gradient function, this method produces the same gradient values at sample locations as if gradients were computed at unit voxel distances from the sample point [S1].

## Subtopics

### [Texture-Mapped Volume Rendering with CPU Shading](3423-texture-mapped-volume-rendering-with-cpu-shading.md)
This method renders regular volumetric grids by leveraging 3D texture mapping hardware to resample density and pre-computed gradients, then transferring the raw sample data to main memory for CPU-based Phong shading and compositing. It solves the problem of view-dependent shading in texture-mapped rendering by avoiding the need to re-shade and reload textures for every frame change, and by circumventing the infeasibly large lookup tables required by post-texturing gradient indexing. Unlike the Planes Method, which may rely on hardware blending for final compositing without explicit CPU shading, this method explicitly offloads the lighting calculation to the CPU to support arbitrary dynamic lights and non-linear transfer functions without hardware interpolation artifacts.

### [Image-Parallel Ray Casting via 2D Texture Mapping](5854-image-parallel-ray-casting-via-2d-texture-mapping.md)
Image-Parallel Ray Casting via 2D Texture Mapping is an object-space volume rendering technique that accelerates direct volume rendering by projecting parallel slices of volumetric data onto the image plane and compositing them using graphics hardware rasterization and blending. It replaces standard ray casting by treating the volume as a stack of 2D texture-mapped rectangles, leveraging hardware-accelerated texture sampling and alpha blending to achieve rendering speeds dependent only on image size rather than scene complexity. Unlike its sibling [Texture-Mapped Volume Rendering with CPU Shading](3423-texture-mapped-volume-rendering-with-cpu-shading.md), this method performs the compositing entirely within the GPU pipeline using standard blending operators.

## Operating characteristics
*   **Time Complexity:** The rendering time is dominated by the texture mapping operations (sequential) and the CPU-based raycasting/compositing (parallelizable) [S1].
*   **Memory Complexity:** Requires storing the volume data in 3D texture memory [S1].
*   **Measured Performance:**
    *   CT scanned Head dataset (bone structures) at 200x200 resolution: 0.44 seconds [S1].
    *   Lobster dataset (sampling resolution 512x512x64): 1.88 seconds [S1].
    *   Opaque Sphere dataset: 1.84 seconds [S1].
    *   Translucent Lobster dataset: 2.78 seconds (12 times faster than Sommer et al.) [S1].
    *   128x128 image of Opaque Sphere with 84 samples per ray: 0.19 seconds [S1].
    *   Lobster dataset (128x128 image, 84 samples per ray): 0.24 seconds [S1].
    *   Time to perform texture mapping of 128x128x84 planes: 0.12 seconds [S1].
*   **Strengths:**
    *   Supports arbitrary parallel and perspective projections in the same time (unlike Shear-Warp) [S1].
    *   No final warp step required, removing a filtering step and resulting in higher image quality [S1].
    *   Supports interactive segmentation/classification (unlike Shear-Warp which requires multiple copies of compressed data structures) [S1].
    *   Allows for CPU-based optimizations like early ray termination and low-opacity skipping [S1].
    *   Produces high-quality Phong shading with specular highlights [S1].
*   **Limitations:**
    *   The parallel speedup is bounded by the time required for sequential texture mapping $T_t$, as there is only one graphics pipeline [S1].
    *   Limited by available texture memory for large volumes [S1].
    *   Slower than methods that do not employ directional shading (e.g., Cabral et al.) [S1].
*   **When to use:** When high-quality view-dependent shading (Phong) is required, interactive classification is needed, or perspective projections are used, and hardware supports 3D texture mapping [S1].
*   **When not to use:** When maximum frame rate is critical and shading quality is secondary, or when the volume size exceeds available texture memory [S1].

## Background
The Planes Method was developed for use on Silicon Graphics Onyx 2 workstations with Infinite Reality graphics hardware [S1]. It was presented as an alternative to shear-warp and other texture-mapped methods to achieve high-quality, accurately shaded volume rendering at interactive frame rates [S1]. The method builds on the concept of using 3D texture mapping hardware for resampling but diverges by performing the compositing and shading on the CPU to leverage software optimizations [S1].

## Sources
[S1] High-Quality Volume Rendering Using Texture Mapping ..., https://cfcs.pku.edu.cn/baoquan/docs/20180622110034067331.pdf, fetched 2026-10-27
