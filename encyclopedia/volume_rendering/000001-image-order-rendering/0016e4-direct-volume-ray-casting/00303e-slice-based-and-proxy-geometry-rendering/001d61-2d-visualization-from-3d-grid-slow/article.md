# 2D Visualization from 3D Grid (Slow)

Also called: 2D Slice Compositing, Direct Volume Rendering (DVR), Slice Projection, Slice-Based Volume Rendering (Back-to-Front), Volume Visualization, matRad_rayTracing Function

## Overview
This method generates a 2D projection of a 3D scalar grid by processing the volume data as a series of 2D slices, applying a transfer function to each slice, and compositing them into a 2D image using the over operator [S4, unofficial]. It solves the problem of visualizing medical/scientific data and natural effects like fluids, smoke, and fire without extracting any surface geometry [S3, unofficial]. It differs from its sibling [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md) by performing explicit per-slice compositing in software (or CPU-side) rather than leveraging hardware-accelerated texture sampling and fragment shaders for ray casting [S4, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume $s: \mathbb{R}^3 \rightarrow \mathbb{R}$, discretely sampled on a regular 3D grid where each voxel contains scalar values (e.g., density, opacity) [S3, unofficial]. Additional inputs may include steering information, CT data, voxel indices, and rotation coordinates for specific applications like dose distribution visualization [S1].
*   **Output:** A 2D image with shape matching the camera grid dimensions, typically normalized to $[0.0, 1.0]$ and converted to uint8 [S4, S2, unofficial].

**Data Structures:**
*   **Volume Grid:** A 3D array of voxels. Each voxel holds scalar data. In some implementations, this is represented as a tensor or cube [S1][S3, unofficial].
*   **Camera Grid:** A structure defining the sequence of 2D slices to be processed. The order of iteration through this grid determines the compositing direction (back-to-front or front-to-back) [S4, pitfall, unofficial].
*   **Transfer Function:** A mapping from scalar input values to RGBA output. In the specific implementation cited, this uses Gaussian mixtures [S4, unofficial].

**Parameters:**
*   **Transfer Function Coefficients:** The specific Gaussian mixture parameters for the cited implementation are:
    *   Red: $r = 1.0 \cdot e^{-\frac{(x - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x - 3.0)^2}{0.1}} + 0.1 \cdot e^{-\frac{(x + 3.0)^2}{0.5}}$
    *   Green: $g = 1.0 \cdot e^{-\frac{(x - 9.0)^2}{1.0}} + 1.0 \cdot e^{-\frac{(x - 3.0)^2}{0.1}} + 0.1 \cdot e^{-\frac{(x + 3.0)^2}{0.5}}$
    *   Blue: $b = 0.1 \cdot e^{-\frac{(x - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x - 3.0)^2}{0.1}} + 1.0 \cdot e^{-\frac{(x + 3.0)^2}{0.5}}$
    *   Alpha: $a = 0.6 \cdot e^{-\frac{(x - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x - 3.0)^2}{0.1}} + 0.01 \cdot e^{-\frac{(x + 3.0)^2}{0.5}}$
    *   Where $x$ is the logarithm of the slice data [S4, unofficial].
*   **Viewing Angle:** Rotation angle $\theta = \frac{\pi}{2} \cdot \frac{i}{N_{angles}}$ around the Y/Z axis, applied via rotation matrices [S4, unofficial].
*   **Lateral Cutoff:** Optional parameter for ray tracing boundaries [S1].
*   **Transparency Threshold:** Optional threshold below which alpha is set to 0.0 [S2, unofficial].

**Steps:**
1.  **Initialize Output:** Create an output image buffer of zeros with dimensions matching the camera grid: `image = np.zeros((camera_grid.shape[1], camera_grid.shape[2], 3))` [S4, unofficial].
2.  **Iterate Slices:** Loop through each slice in the `camera_grid`. The order of this loop is critical: if `camera_grid` is ordered back-to-front relative to the viewer, this implements front-to-back compositing; if front-to-back, it implements back-to-front [S4, pitfall, unofficial].
3.  **Transform Data:** Apply a logarithmic transformation to the slice data: `log_data = np.log(dataslice)` [S4, unofficial].
4.  **Apply Transfer Function:** Map the log-transformed data to RGBA values using the Gaussian mixture transfer function defined above [S4, unofficial].
5.  **Composite:** For each RGB channel, composite the current slice onto the accumulated image using the over operator:
    $$I_{new} = a \cdot C + (1 - a) \cdot I_{old}$$
    Where $I_{new}$ is the new pixel value, $a$ is the alpha from the transfer function, $C$ is the color from the transfer function, and $I_{old}$ is the current accumulated pixel value [S4, unofficial].
6.  **Clip and Convert:** Clip the final image values to $[0.0, 1.0]$ and convert to uint8 format [S4, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_transfer_function(x):
    """
    Applies the specific Gaussian mixture transfer function.
    x: input scalar value (log-transformed)
    Returns: r, g, b, a
    """
    # Red channel
    r = (1.0 * np.exp(-((x - 9.0)**2) / 1.0) +
         0.1 * np.exp(-((x - 3.0)**2) / 0.1) +
         0.1 * np.exp(-((x + 3.0)**2) / 0.5))
    
    # Green channel
    g = (1.0 * np.exp(-((x - 9.0)**2) / 1.0) +
         1.0 * np.exp(-((x - 3.0)**2) / 0.1) +
         0.1 * np.exp(-((x + 3.0)**2) / 0.5))
    
    # Blue channel
    b = (0.1 * np.exp(-((x - 9.0)**2) / 1.0) +
         0.1 * np.exp(-((x - 3.0)**2) / 0.1) +
         1.0 * np.exp(-((x + 3.0)**2) / 0.5))
    
    # Alpha channel
    a = (0.6 * np.exp(-((x - 9.0)**2) / 1.0) +
         0.1 * np.exp(-((x - 3.0)**2) / 0.1) +
         0.01 * np.exp(-((x + 3.0)**2) / 0.5))
    
    return r, g, b, a

def render_2d_from_3d_grid(volume_data, camera_grid):
    """
    Renders a 2D image from a 3D grid by compositing slices.
    
    Args:
        volume_data: 3D numpy array representing the scalar volume.
        camera_grid: List or array of 2D slices to process in order.
        
    Returns:
        2D numpy array (H, W, 3) with values in [0, 255] uint8.
    """
    # Initialize output image
    # Assuming camera_grid elements have shape (H, W)
    if len(camera_grid) == 0:
        return np.zeros((0, 0, 3), dtype=np.uint8)
        
    h, w = camera_grid[0].shape
    image = np.zeros((h, w, 3), dtype=np.float32)
    
    for dataslice in camera_grid:
        # Step 1: Log transform
        log_data = np.log(dataslice + 1e-8) # Add epsilon to avoid log(0)
        
        # Step 2: Apply transfer function
        r, g, b, a = gaussian_transfer_function(log_data)
        
        # Clip alpha to [0, 1] if necessary, though Gaussian mixtures are positive
        a = np.clip(a, 0.0, 1.0)
        
        # Step 3: Composite using over operator
        # image[:,:,0] is Red channel, etc.
        image[:,:,0] = a * r + (1.0 - a) * image[:,:,0]
        image[:,:,1] = a * g + (1.0 - a) * image[:,:,1]
        image[:,:,2] = a * b + (1.0 - a) * image[:,:,2]
        
    # Step 4: Clip and convert
    image = np.clip(image, 0.0, 1.0)
    image_uint8 = (image * 255).astype(np.uint8)
    
    return image_uint8
```

**Edge Cases and Pitfalls:**
*   **Compositing Order:** The correctness of the result depends heavily on the order of slices in `camera_grid`. If the grid is not sorted by depth relative to the viewer, the compositing will be incorrect (e.g., front objects occluding back objects incorrectly) [S4, pitfall, unofficial].
*   **Logarithm of Zero:** Applying `np.log()` to data containing zero values will result in `-inf`. A small epsilon should be added to the data before logging [G].
*   **Precision:** Accumulating many slices can lead to floating-point precision issues if the number of slices is very large. Using `float32` is standard, but `float64` may be needed for high dynamic range or very deep volumes [G].

## Subtopics

### [Grid Visualization with Thresholding](1515-grid-visualization-with-thresholding.md)
This method visualizes a 3D scalar grid by filtering cells based on opacity and gradient magnitude thresholds, then rendering the surviving cells as a 3D scatter plot. It solves the problem of visualizing sparse or noisy volumetric data by discarding low-confidence or irrelevant voxels. It differs from its parent [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md) by explicitly applying thresholding and cleaning steps before plotting, and from its sibling [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md) by operating on a CPU-side scatter plot representation rather than a GPU-based ray-casting table.

### [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md)
This method computes a 3D pre-integration table for volume rendering by rasterizing 2D slices on the GPU, where each pixel represents a specific triplet of scalar values $(s_f, s_m, s_b)$ along a ray. It solves the problem of efficiently evaluating the volume rendering integral for transfer functions defined on 8-bit scalar domains by pre-computing the integration results into a texture. It is distinguished from its parent method, [2D Visualization from 3D Grid (Slow)](1507-2d-visualization-from-3d-grid-slow.md), by utilizing GPU rasterization and pre-integration to accelerate the evaluation of the volume rendering equation, rather than performing per-ray sampling in software or on the CPU.

## Operating characteristics
*   **Time Complexity:** $O(N_s \cdot H \cdot W)$, where $N_s$ is the number of slices, and $H \times W$ is the image resolution. This is slow compared to GPU-accelerated methods because it performs per-pixel compositing in software [S4, S2, unofficial].
*   **Memory Complexity:** $O(H \cdot W)$ for the output image buffer, plus $O(V)$ for the volume data, where $V$ is the number of voxels.
*   **Performance:** The cited implementation is designed for visualization and debugging rather than real-time rendering. It is suitable for small to medium-sized volumes or when GPU resources are unavailable [S4, S2, unofficial].
*   **Strengths:** Simple to implement, easy to debug, flexible transfer function application [S4, unofficial].
*   **Limitations:** Slow for large volumes, no hardware acceleration, sensitive to compositing order [S4, pitfall, unofficial].
*   **When to use:** When implementing a reference solution, debugging transfer functions, or when GPU programming is not feasible [S4, S2, unofficial].
*   **When not to use:** For real-time applications or large-scale volumes where performance is critical [S2, unofficial].

## Background
The concept of volume visualization as a 2D projection of 3D data without surface extraction was introduced by Eric Shaffer in lecture materials for CS 519: Scientific Visualization at the University of Illinois Urbana-Champaign [S3, unofficial]. The specific implementation using Gaussian mixture transfer functions and slice-based compositing is found in the `Parallel_Volume_Rendering` repository [S4, unofficial]. The `matRad` project provides a related ray-tracing function for dose distribution visualization, which shares the goal of 2D projection from 3D grids but uses a different ray-casting approach [S1]. The `Plenoxels` project also utilizes slice-based sampling and compositing for neural radiance fields, highlighting the method's relevance in modern rendering contexts [S2, unofficial].

## Sources
[S1] rayTracing — matRad 3.2.3 documentation, https://matrad.readthedocs.io/en/stable/api/raytracing.html, fetched 2026-10-25
[S2] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S4] SREESUBBARAYUDU/Parallel_Volume_Rendering, https://github.com/SREESUBBARAYUDU/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
