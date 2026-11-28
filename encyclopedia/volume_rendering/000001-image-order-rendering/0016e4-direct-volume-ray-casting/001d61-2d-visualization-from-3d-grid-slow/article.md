# 2D Visualization from 3D Grid (Slow)

Also called: 2D Slice Compositing, Direct Volume Rendering (DVR), Slice Projection, Slice-Based Volume Rendering (Back-to-Front), matRad_rayTracing Function

## Overview
This method generates a 2D projection of a 3D scalar volume by processing the data as a series of 2D slices, applying a transfer function to each slice, and compositing them into a 2D image using the over operator [S4, unofficial]. It solves the problem of visualizing volumetric data without extracting surface geometry [S3, unofficial]. It sets itself apart from its siblings by explicitly iterating through slices in a software loop and performing per-slice compositing, often relying on specific data structures like the `camera_grid` for ordering [S4, unofficial].

## How it works
The method operates primarily as a software-based iterative compositing loop [S4, unofficial], with a specialized ray-tracing depth calculation variant in matRad [S1].

### Inputs and Outputs
*   **Input Volume ($V$):** A scalar field $s: \mathbb{R}^3 \rightarrow \mathbb{R}$, discretely sampled on a regular 3D grid [S3, unofficial].
*   **Camera/Grid Configuration:** A set of viewing parameters defining the projection plane [G]. In the iterative model, this is represented as a `camera_grid` containing slice data ordered for compositing [S4, unofficial]. In the ray-tracing model, this includes steering information (`stfElement`), rotation coordinates (`rot_coordsV`), and lateral cutoffs [S1].
*   **Transfer Function ($TF$):** A mapping from scalar values to RGBA color and opacity [G]. The notes specify a Gaussian mixture model applied to the logarithm of the input data [S4, unofficial].
*   **Output Image ($I$):** A 2D image with dimensions matching the camera grid, containing the composited RGB values [S4, unofficial].

### Data Structures
1.  **Volume Grid:** A 3D array of scalars. In the ray-tracing context, linear voxel indices $V$ and rotation coordinates `rot_coordsV` are used to map 3D space to the beam's eye view [S1].
2.  **Camera Grid:** A list or array of 2D slices (`dataslice`) [S4, unofficial]. The order of iteration through this grid determines the compositing direction (back-to-front or front-to-back) [S4, unofficial].
3.  **Radiological Depth Cube (`radDepthCube`):** In the matRad implementation, a 3D structure storing radiological depth values computed from CT data and voxel indices [S1].

### Transfer Function Formulas
The transfer function uses Gaussian mixtures [S4, unofficial]. The input scalar $x$ is first transformed using the natural logarithm: $x' = \ln(x)$ [S4, unofficial]. The resulting Red ($r$), Green ($g$), Blue ($b$), and Alpha ($a$) components are calculated as follows [S4, unofficial]:

$$
r = 1.0 \cdot e^{-\frac{(x' - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x' - 3.0)^2}{0.1}} + 0.1 \cdot e^{-\frac{(x' + 3.0)^2}{0.5}}
$$

$$
g = 1.0 \cdot e^{-\frac{(x' - 9.0)^2}{1.0}} + 1.0 \cdot e^{-\frac{(x' - 3.0)^2}{0.1}} + 0.1 \cdot e^{-\frac{(x' + 3.0)^2}{0.5}}
$$

$$
b = 0.1 \cdot e^{-\frac{(x' - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x' - 3.0)^2}{0.1}} + 1.0 \cdot e^{-\frac{(x' + 3.0)^2}{0.5}}
$$

$$
a = 0.6 \cdot e^{-\frac{(x' - 9.0)^2}{1.0}} + 0.1 \cdot e^{-\frac{(x' - 3.0)^2}{0.1}} + 0.01 \cdot e^{-\frac{(x' + 3.0)^2}{0.5}}
$$

### Algorithm Steps
The following steps describe the software-based implementation found in [S4, unofficial].

1.  **Initialization:** Initialize the output image $I$ to zeros with shape matching the camera grid dimensions:
    ```python
    image = np.zeros((camera_grid.shape[1], camera_grid.shape[2], 3))
    ```
2.  **Iteration:** Iterate through each slice in the `camera_grid` [S4, unofficial].
    ```python
    for dataslice in camera_grid:
    ```
3.  **Transfer Function Application:** Apply the transfer function to the logarithm of the slice data to obtain RGBA values for each pixel in the slice:
    ```python
    r, g, b, a = transfer_function(np.log(dataslice))
    ```
4.  **Compositing:** Composite the slice onto the accumulated image using the standard over operator for each RGB channel [S4, unofficial]. For the red channel (and similarly for green and blue):
    ```python
    image[:, :, 0] = a * r + (1 - a) * image[:, :, 0]
    ```
    If `a` is 0, the slice is transparent and does not affect the background [G]. If `a` is 1, the slice completely occludes the background [G].
5.  **Clipping:** Clip the final image values to the range $[0.0, 1.0]$ to handle any numerical overflow:
    ```python
    image = np.clip(image, 0.0, 1.0)
    ```

### Parameters
*   **Logarithmic Transformation:** The input data is transformed using `np.log()` before applying the transfer function [S4, unofficial].
*   **Viewing Angle:** The camera view can be rotated by an angle $\theta = \frac{\pi}{2} \cdot \frac{i}{N_{angles}}$ around the Y/Z axis using rotation matrices [S4, unofficial]:
    $$
    q_{yR} = q_y \cos(\theta) - q_z \sin(\theta)
    $$
    $$
    q_{zR} = q_y \sin(\theta) + q_z \cos(\theta)
    $$
*   **Transparency Threshold:** In some implementations, the alpha channel (last dimension) of the grid cells tensor is thresholded below a `transparency_threshold` to 0.0 [S2, unofficial].

### Edge Cases and Pitfalls
*   **Compositing Order:** The loop `for dataslice in camera_grid:` implies a specific order [S4, unofficial]. If `camera_grid` is ordered from back to front relative to the viewer, this implements front-to-back compositing (standard for alpha blending) [S4, unofficial]. If ordered front-to-back, it implements back-to-front compositing [S4, unofficial]. The code does not explicitly sort slices by depth, relying on the data structure order [S4, unofficial]. Incorrect ordering leads to visual artifacts where distant objects appear in front of closer ones [G].
*   **Logarithm of Zero/Negative:** Applying `np.log()` to zero or negative voxel values will result in `-inf` or `nan` [G]. The transfer function must handle these cases, or the input data must be shifted/clipped to positive values before logging [G].

## Subtopics

### [Grid Visualization with Thresholding](1515-grid-visualization-with-thresholding.md)
A sparse visualization technique that filters a 3D grid tensor based on opacity and gradient magnitude thresholds, then renders the surviving cells as a 3D scatter plot. It avoids the computational overhead of ray casting or volume integration, making it suitable for rapid inspection of sparse structures. It sets itself apart from its sibling [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md) by avoiding precomputed lookup tables and instead performing direct scalar filtering and point plotting, and from its parent by explicitly leveraging gradient magnitude for structural filtering and outputting a 3D point cloud rather than 2D slices.

### [3D Table Computation on GPU](2083-3d-table-computation-on-gpu.md)
This method computes a 3D pre-integration table for volume rendering by rasterizing 2D slices on the GPU, where each pixel represents a specific triplet of scalar values $(s_f, s_m, s_b)$ along a ray. It solves the problem of efficiently evaluating the volume rendering integral for transfer functions defined on 8-bit scalar domains by pre-computing the integration results into a texture. It is distinguished from its parent by utilizing GPU rasterization and pre-integration to accelerate the evaluation of the volume rendering equation, rather than performing per-ray sampling in software or on the CPU.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The concept of Direct Volume Rendering (DVR) as a 2D projection of 3D data without surface extraction was established in the early days of scientific visualization [S3, unofficial]. The specific iterative slice-compositing approach described here is a software implementation of the volume rendering integral, often used in research contexts or specialized medical planning tools like matRad [S1]. The matRad implementation specifically targets radiation therapy planning, visualizing dose distributions on CT data [S1].

## Sources
[S1] rayTracing — matRad 3.2.3 documentation, https://matrad.readthedocs.io/en/stable/api/raytracing.html, fetched 2026-10-25
[S2] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S4] SREESUBBARAYUDU/Parallel_Volume_Rendering, https://github.com/SREESUBBARAYUDU/Parallel_Volume_Rendering, fetched 2026-10-27 (unofficial)
