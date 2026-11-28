# Silhouette Detection via Laplacian Filter

Also called: Laplacian Filter Silhouette Detection

## Overview
This method detects object silhouettes in image space to prevent advection particles from flowing across geometric boundaries, which would otherwise render the surface's geometry and topology unrecognizable [S1, book][S2, unofficial]. It builds on [LOG-based Silhouette Detection](6322-log-based-silhouette-detection.md) by applying a discrete Laplacian filter to depth values followed by thresholding [S1, book][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Depth values $d(x,y)$ for each pixel $(x,y)$ in the image space.
*   **Output:** A silhouette image $e(x,y)$ stored in the red channel of an output texture. The value $s = e(x,y)$ represents the silhouette strength, typically in the range $[0, 1]$ [S2, unofficial].

**Data Structures:**
*   **Silhouette Texture:** A 2D texture where the red channel stores the silhouette image $e_{x,y}$. Other channels are not specified for this step [S1, book].

**Algorithm:**
1.  **Convolution:** Apply a standard three-by-three Laplacian filter convolution mask to the depth values [S1, book].
2.  **Thresholding:** Threshold the result to determine silhouette pixels [S1, book].
3.  **Storage:** Store the silhouette image $e_{x,y}$ in the red color channel of the output texture [S1, book].

**Kernel Definition:**
The discrete 3x3 Laplacian filter kernel $D_{xy}^2$ is defined as [S1, book][S2, unofficial]:
$$ D_{xy}^2 = \begin{bmatrix} 0 & 1 & 0 \\ 1 & -4 & 1 \\ 0 & 1 & 0 \end{bmatrix} $$
This approximates the continuous Laplace operator $\Delta f = \frac{\partial^2 f}{\partial x^2} + \frac{\partial^2 f}{\partial y^2}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_silhouette(depth_buffer, threshold):
    """
    Computes silhouette mask using Laplacian filter on depth buffer.
    
    Args:
        depth_buffer: 2D list of depth values d(x,y).
        threshold: Scalar threshold for silhouette detection.
        
    Returns:
        silhouette_texture: 2D list where values represent silhouette mask.
    """
    height = len(depth_buffer)
    width = len(depth_buffer[0])
    silhouette = [[0.0 for _ in range(width)] for _ in range(height)]
    kernel = [[0, 1, 0], [1, -4, 1], [0, 1, 0]]
    
    for y in range(1, height - 1):
        for x in range(1, width - 1):
            laplacian_value = 0.0
            for ky in range(3):
                for kx in range(3):
                    laplacian_value += kernel[ky][kx] * depth_buffer[y - 1 + ky][x - 1 + kx]
            if laplacian_value > threshold:
                silhouette[y][x] = 1.0
    # Boundary handling not specified in sources [G]; edge pixels remain 0.0
    return silhouette
```

**Parameters:**
*   **Threshold:** The specific threshold value is not given in the sources.

**Edge Cases and Pitfalls:**
*   **Advection Across Boundaries:** Without silhouette detection, tensor advection leads to a constant flow of "particles" across surface boundaries, making the surface's geometry and topology unrecognizable [S1, book][S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This method is part of image-space tensor field visualization techniques, specifically used in Line Integral Convolution (LIC)-like methods for fabric-like visualization on arbitrary surfaces [S1, book][S2, unofficial]. It addresses the problem of advection over geometric boundaries identified in earlier work [LJH03] cited in [S1, book][S2, unofficial].

## Sources
[S1] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
[S2] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
