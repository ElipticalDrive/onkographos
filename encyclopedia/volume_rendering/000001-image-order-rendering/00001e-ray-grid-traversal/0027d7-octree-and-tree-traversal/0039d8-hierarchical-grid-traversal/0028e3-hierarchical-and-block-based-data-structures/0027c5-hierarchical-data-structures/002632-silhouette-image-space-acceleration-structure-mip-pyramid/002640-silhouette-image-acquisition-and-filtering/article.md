# Silhouette Image Acquisition and Filtering


## Overview
This method generates calibrated binary silhouette images from raw camera sensor data to define a visual hull surface $\mathcal{S}(\mathbf{p}) = 0.5$ [S1]. It solves the problem of converting unprocessed Bayer pattern data into smooth occupancy masks suitable for hierarchical acceleration structures [S1]. It is distinguished from standard silhouette extraction by integrating radial lens distortion correction, demosaicing, and chroma keying into a single GPU compute kernel, followed by separable Gaussian filtering optimized for shared memory bandwidth [S1].

## How it works
**Inputs:** Raw 8-bit Bayer pattern image data from $n$ camera views [S1].
**Outputs:** $n$ filtered silhouette images stored as 2D textures, where foreground pixels have value 0 and background pixels have value 1 [S1].

**Data Structures:**
*   **Input Texture Array:** 2D texture array storing raw 8-bit Bayer data per camera view [S1].
*   **Output Texture Array:** 2D texture array storing the resulting binary silhouette values [S1].

**Steps:**
1.  **Upload Data:** Transfer raw Bayer pattern data to the GPU as a 2D texture array [S1].
2.  **Acquisition Kernel:** Launch a single compute kernel with one thread per pixel. Each thread simultaneously performs:
    a.  **Distortion Correction:** Maps pixel coordinates to ideal undistorted coordinates using inverse radial distortion [S1].
    b.  **Demosaicing:** Interpolates full intensity values from the Bayer color filter array [S1].
    c.  **Chroma Keying:** Segments foreground and background based on color/intensity thresholds, assigning 0 to foreground pixels and 1 to background pixels [S1].
3.  **Filtering:** Apply a separable Gaussian kernel with unit integral to the resulting binary images [S1].
    *   **Implementation:** Use a thread block per row/column strategy so the entire image row or column fits in shared memory, maximizing convolution performance on the GPU [S1].
    *   **Effect:** Increasing the kernel width increases blur, resulting in a smoother surface $\mathcal{S}(\mathbf{p})$ [S1]. The gradient $\nabla \mathcal{S}(\mathbf{p})$ provides a smooth normal field over $\mathcal{S}(\mathbf{p}) = 0.5$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def apply_radial_distortion(u, v, k1, k2):
    # Inverse radial distortion to map pixel to ideal coordinates
    r2 = u*u + v*v
    factor = 1.0 / (1.0 + k1 * r2 + k2 * r2 * r2)
    return u * factor, v * factor

def demosaic_bayer(bayer_tex, u, v):
    # Bilinear interpolation from Bayer grid samples
    x0, y0 = math.floor(u), math.floor(v)
    x1, y1 = x0 + 1, y0 + 1
    fx, fy = u - x0, v - y0
    val00 = bayer_tex[x0, y0]
    val01 = bayer_tex[x0, y1]
    val10 = bayer_tex[x1, y0]
    val11 = bayer_tex[x1, y1]
    return (val00 * (1-fx) * (1-fy) + val01 * (1-fx) * fy +
            val10 * fx * (1-fy) + val11 * fx * fy)

def chroma_key(intensity, threshold=0.5):
    return 0.0 if intensity < threshold else 1.0

def gaussian_filter_1d(data, kernel_width):
    # Separable 1D Gaussian with unit integral
    sigma = kernel_width / 2.0
    kernel = []
    for i in range(-int(kernel_width), int(kernel_width) + 1):
        kernel.append(math.exp(-0.5 * (i/sigma)**2))
    norm = sum(kernel)
    kernel = [k/norm for k in kernel]
    result = []
    for i in range(len(data)):
        acc = 0.0
        for j, w in enumerate(kernel):
            idx = i + j - len(kernel)//2
            idx = max(0, min(len(data)-1, idx))
            acc += data[idx] * w
        result.append(acc)
    return result

def silhouette_pipeline(bayer_texture, camera_params, kernel_width):
    # Step 1 & 2: Acquisition Kernel (per pixel)
    silhouette_images = []
    for view in range(len(bayer_texture)):
        img = bayer_texture[view]
        h, w = img.shape
        sil_img = [[0.0]*w for _ in range(h)]
        for y in range(h):
            for x in range(w):
                u_ideal, v_ideal = apply_radial_distortion(x - w/2, y - h/2, camera_params.k1, camera_params.k2)
                intensity = demosaic_bayer(img, u_ideal + w/2, v_ideal + h/2)
                sil_img[y][x] = chroma_key(intensity)
        silhouette_images.append(sil_img)

    # Step 3: Filtering (Separable Gaussian)
    filtered_images = []
    for sil_img in silhouette_images:
        # Horizontal pass
        h_pass = [gaussian_filter_1d(row, kernel_width) for row in sil_img]
        # Vertical pass
        v_pass = [gaussian_filter_1d([h_pass[y][x] for y in range(len(h_pass))], kernel_width) for x in range(len(h_pass[0]))]
        # Transpose back
        filtered = [[v_pass[x][y] for x in range(len(v_pass))] for y in range(len(v_pass[0]))]
        filtered_images.append(filtered)
    return filtered_images
```

**Parameters:**
*   **Gaussian Kernel Width:** Controls the amount of blur. Larger values produce smoother surfaces $\mathcal{S}(\mathbf{p})$ [S1]. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Using 8-bit Bayer data reduces camera bus and CPU/GPU bandwidth [S1].
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** If chroma key thresholds are misaligned with the camera's color space, segmentation fails [G].

## Subtopics

### [Visual Hull Oracle Kernel](5503-visual-hull-oracle-kernel.md)
The Visual Hull Oracle Kernel is a parallel processing unit that performs binary culling of voxels to determine if they intersect the visual hull surface $\mathcal{S}(\mathbf{p}) = 0.5$. It solves the problem of efficiently identifying which voxels in a sparse voxel grid contribute to the final volume by leveraging hierarchical mip pyramids over silhouette images to compute conservative bounds on voxel coverage. It is distinguished from sibling methods by its specific use of component-wise multiplication of conservative bounds across multiple camera views to make a definitive KEEP or CULL decision, building on the silhouette acquisition and filtering techniques described in [Silhouette Image Acquisition and Filtering](3754-silhouette-image-acquisition-and-filtering.md).

### [Pixel separation for bright foreground](9092-pixel-separation-for-bright-foreground.md)
A technique to correct incorrect blurring of bright pixels in low depths (foreground) against dark backgrounds by splitting the input into two separate scattering images: one for separated bright pixels ($L_{ssc}$) and one for the rest ($L_{sc}$).

### Slab Silhouette Maps with Early Depth Test
A technique exploiting slab silhouette maps and an early depth test to skip empty spaces along the direction normal to the texture slice [S2]. It is distinguished by its ability to bypass non-transparent regions efficiently, improving rendering performance for large volumes [S2]. Choose this method when rendering large, sparse volumes with significant empty space along the view direction [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels for acquisition and filtering [S1].
*   **Memory Complexity:** Proportional to the size of the input and output texture arrays [S1].
*   **Performance:** High performance on GPU due to parallel processing and shared memory optimization for the Gaussian filter [S1].
*   **Strengths:** Reduces data bandwidth by using raw Bayer data; produces smooth surfaces suitable for gradient-based normal estimation [S1].
*   **Limitations:** Smoothing may reduce geometric accuracy if the kernel width is too large [S1].
*   **When to use:** When real-time or high-resolution silhouette generation is needed for visual hull reconstruction [S1].
*   **When not to use:** When exact geometric boundaries are required without smoothing [S1].

## Background
The method is detailed in a paper on real-time high-resolution sparse voxelization with application to image-based modeling [S1]. It leverages GPU compute capabilities to perform complex image processing tasks (distortion correction, demosaicing, filtering) in a single pass, optimizing for modern graphics hardware [S1].

## Sources
[S1] Real-time high-resolution sparse voxelization with application to image-based modeling, https://doi.org/10.1145/2492045.2492053, fetched 2026-11-01
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
