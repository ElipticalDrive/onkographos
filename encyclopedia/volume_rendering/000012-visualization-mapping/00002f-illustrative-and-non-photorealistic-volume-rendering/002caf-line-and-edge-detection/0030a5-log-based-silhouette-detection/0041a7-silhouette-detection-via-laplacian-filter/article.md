# Silhouette Detection via Laplacian Filter

Also called: Laplacian Filter Silhouette Detection

## Overview
This method detects object silhouettes in image space to prevent advection particles from flowing across geometric boundaries, which would otherwise render the surface's geometry and topology unrecognizable [S1, book][S2, unofficial]. It replaces exhaustive 3x3x3 subspace searches used in parent methods by applying a discrete Laplacian filter to depth values followed by thresholding [S1, book][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Depth values $d(x,y)$ for each pixel $(x,y)$ in the image space.
*   **Output:** A silhouette image $e(x,y)$ stored in the red channel of an output texture. The value $s = e(x,y)$ represents the silhouette strength, typically in the range $[0, 1]$ [S2, unofficial].

**Data Structures:**
*   **Silhouette Texture:** A 2D texture where the red channel stores the binary or thresholded silhouette mask $e_{x,y}$. Other channels are not specified for this specific step [S1, book].

**Algorithm:**
1.  **Convolution:** Apply a standard 3x3 Laplacian filter kernel to the depth buffer. For each pixel $(x,y)$, compute the discrete Laplacian approximation:
    $$ L(x,y) = \sum_{i=-1}^{1} \sum_{j=-1}^{1} K_{i,j} \cdot d(x+i, y+j) $$
    where $K$ is the kernel defined below.
2.  **Thresholding:** Compare the result $L(x,y)$ against a threshold to determine if the pixel lies on a silhouette. Pixels exceeding the threshold are marked as silhouette pixels.
3.  **Storage:** Store the resulting silhouette value $s$ in the red channel of the output texture for pixel $(x,y)$.

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
        depth_buffer: 2D array of depth values d(x,y).
        threshold: Scalar threshold for silhouette detection.
        
    Returns:
        silhouette_texture: 2D array where red channel contains silhouette values.
    """
    height, width = depth_buffer.shape
    silhouette = np.zeros((height, width))
    
    # Kernel definition
    kernel = np.array([[0, 1, 0],
                       [1, -4, 1],
                       [0, 1, 0]])
    
    for y in range(1, height - 1):
        for x in range(1, width - 1):
            # Extract 3x3 neighborhood
            neighborhood = depth_buffer[y-1:y+2, x-1:x+2]
            
            # Apply convolution
            laplacian_value = np.sum(neighborhood * kernel)
            
            # Thresholding
            if laplacian_value > threshold:
                silhouette[y, x] = 1.0
            else:
                silhouette[y, x] = 0.0
                
    return silhouette
```
*Note: Boundary conditions for pixels at the edge of the image are not specified in the sources [G]. Typically, these are handled by clamping, mirroring, or ignoring edge pixels.*

**Parameters:**
*   **Threshold:** The specific value used to binarize the Laplacian response is not given in the sources. It must be determined empirically based on the depth range and noise characteristics of the scene [S1, book][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Advection Across Boundaries:** Without this detection, tensor advection leads to a constant flow of "particles" across surface boundaries, making geometry and topology unrecognizable [S1, book][S2, unofficial].
*   **Depth Discontinuities:** The method relies on depth discontinuities. Smooth gradients may not trigger the filter, while noise might cause false positives if the threshold is too low.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, as it involves a fixed-size convolution per pixel.
*   **Memory Complexity:** $O(N)$ to store the output silhouette texture.
*   **Strengths:** Simple to implement; computationally efficient compared to volumetric or exhaustive subspace searches; effective at stopping advection across visible boundaries.
*   **Limitations:** Image-space only; depends on accurate depth buffer; may miss silhouettes if depth discontinuities are subtle or obscured.

## Background
This method is part of image-space tensor field visualization techniques, specifically used in Line Integral Convolution (LIC)-like methods for fabric-like visualization on arbitrary surfaces [S1, book][S2, unofficial]. It addresses the problem of advection over geometric boundaries identified in earlier work [LJH03] cited in [S1, book][S2, unofficial].

## Sources
[S1] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
[S2] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
