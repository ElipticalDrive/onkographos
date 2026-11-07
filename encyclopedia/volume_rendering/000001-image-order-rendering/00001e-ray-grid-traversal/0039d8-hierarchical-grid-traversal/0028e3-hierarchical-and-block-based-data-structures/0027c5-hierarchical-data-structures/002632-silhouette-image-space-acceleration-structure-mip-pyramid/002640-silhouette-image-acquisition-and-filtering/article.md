# Silhouette Image Acquisition and Filtering

Also called: Chroma-key silhouette generation, Gaussian-filtered silhouettes

## Overview
This method generates calibrated binary silhouette images from raw camera data to define a visual hull surface $S(\mathbf{p}) = 0.5$ [S1]. It solves the problem of converting unprocessed sensor data into smooth, geometrically accurate occupancy masks suitable for hierarchical acceleration structures [S1]. It is distinguished from standard silhouette extraction by its integration of lens distortion correction, demosaicing, and Gaussian filtering into a single GPU compute kernel, optimizing for bandwidth reduction and surface smoothness [S1].

## How it works
**Inputs:** Raw Bayer pattern image data from $n$ camera views; camera intrinsic and extrinsic parameters; Gaussian kernel width $\sigma$.
**Outputs:** $n$ filtered silhouette images (2D textures), where foreground pixels have value 0 and background pixels have value 1.

**Data Structures:**
*   **Input Texture Array:** 2D texture array storing raw 8-bit Bayer data. This reduces camera bus and CPU/GPU bandwidth compared to full RGB formats [S1].
*   **Output Texture Array:** 2D texture array storing the resulting binary silhouette values.

**Steps:**
1.  **Upload Data:** Transfer raw Bayer pattern data to the GPU as a 2D texture array [S1].
2.  **Compute Kernel Launch:** Launch a single compute kernel with one thread per pixel. The kernel performs three operations simultaneously:
    a.  **Distortion Correction:** Remove radial lens distortion from the pixel coordinates [S1].
    b.  **Demosaicing:** Convert the Bayer pattern to full color or intensity values [S1].
    c.  **Chroma Keying:** Segment foreground and background. Assign value 0 to foreground pixels and value 1 to background pixels [S1].
3.  **Filtering:** Apply a separable Gaussian kernel with unit integral to the resulting binary images [S1].
    *   **Implementation:** Use a thread block per row/column strategy. This allows the entire image row (or column) to fit within shared memory, maximizing convolution performance on the GPU [S1].
    *   **Effect:** Increasing the kernel width increases blur, resulting in a smoother surface $S(\mathbf{p})$ [S1]. The gradient $\nabla S(\mathbf{p})$ provides a smooth normal field over the surface $S(\mathbf{p}) = 0.5$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the single compute kernel per pixel
def silhouette_kernel(input_bayer_texture, output_silhouette_texture, camera_params, kernel_width):
    # 1. Get pixel coordinates
    uv = get_pixel_coordinates()
    
    # 2. Remove radial lens distortion
    # [G] Standard radial distortion model: u_distorted = u * (1 + k1*r^2 + k2*r^4)
    # Inverse distortion is applied to map pixel to ideal coordinates
    uv_ideal = undistort(uv, camera_params.k1, camera_params.k2)
    
    # 3. Demosaic Bayer pattern
    # [G] Bilinear interpolation from neighboring Bayer samples
    intensity = demosaic_bayer(input_bayer_texture, uv_ideal)
    
    # 4. Chroma key segmentation
    # Foreground pixels assigned 0, background pixels assigned 1
    # [G] Thresholding logic depends on specific chroma key color; 
    # assuming intensity threshold or color distance check is performed here.
    if is_foreground(intensity):
        value = 0.0
    else:
        value = 1.0
        
    # 5. Filter (Gaussian)
    # Note: The source describes filtering as a separate step or part of the pipeline.
    # If done in the same kernel, it requires neighbor access.
    # However, the source emphasizes separability and shared memory for performance,
    # suggesting a two-pass approach or a specific block layout.
    # We implement the filtering as a post-process or integrated pass using shared memory.
    
    # For the purpose of this method description, we assume the filtering is applied
    # to the binary result. A separable Gaussian filter is applied.
    # Pass 1: Horizontal
    # Pass 2: Vertical
    # [G] Standard separable Gaussian convolution implementation using shared memory.
    
    output_silhouette_texture[uv] = value
```

**Parameters:**
*   **Gaussian Kernel Width ($\sigma$):** Controls the amount of blur. Larger values produce smoother surfaces $S(\mathbf{p})$ [S1]. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Using 8-bit Bayer data reduces bandwidth but may limit dynamic range compared to 16-bit or 32-bit formats [S1].
*   **Boundaries:** The separable Gaussian filter requires careful handling of image boundaries to avoid artifacts [S1].
*   **Degenerate Input:** If the chroma key fails to segment foreground/background correctly, the silhouette will be inaccurate [S1].

## Subtopics

### Slab Silhouette Maps with Early Depth Test
A technique exploiting slab silhouette maps and an early depth test to skip empty spaces along the direction normal to the texture slice [S2]. It is distinguished by its ability to bypass non-transparent regions efficiently, improving rendering performance for large volumes [S2]. Choose this method when rendering large, sparse volumes with significant empty space along the view direction [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the input images for acquisition and filtering [S1].
*   **Memory Complexity:** Proportional to the size of the input and output texture arrays [S1].
*   **Performance:** High performance on GPU due to parallel processing and shared memory optimization for the Gaussian filter [S1].
*   **Strengths:** Reduces data bandwidth by using raw Bayer data; produces smooth surfaces suitable for gradient-based normal estimation [S1].
*   **Limitations:** Requires GPU compute capability; smoothing may reduce geometric accuracy if the kernel width is too large [S1].
*   **When to use:** When real-time or high-resolution silhouette generation is needed for visual hull reconstruction [S1].
*   **When not to use:** When exact geometric boundaries are required without smoothing [S1].

## Background
Originated in the context of real-time high-resolution sparse voxelization and image-based modeling [S1]. The method leverages GPU compute capabilities to perform complex image processing tasks (distortion correction, demosaicing, filtering) in a single pass, optimizing for modern graphics hardware [S1].

## Sources
[S1] Real-time high-resolution sparse voxelization with application to image-based modeling, https://doi.org/10.1145/2492045.2492053, fetched 2026-11-01
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
