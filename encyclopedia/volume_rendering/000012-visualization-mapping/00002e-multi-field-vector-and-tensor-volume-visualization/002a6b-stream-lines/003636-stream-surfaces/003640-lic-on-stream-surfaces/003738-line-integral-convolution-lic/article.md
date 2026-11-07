# Line Integral Convolution (LIC)
## Overview
Line Integral Convolution (LIC) is a texture-based visualization method that maps a scalar noise field along the streamlines of a vector field to reveal flow direction and structure [S1, unofficial]. Unlike direct streamlines which show discrete paths, LIC produces a continuous texture where the orientation of noise patterns aligns with the local flow, effectively visualizing the entire domain as a coherent image [S1, unofficial]. It is distinct from its parent method, [LIC on Stream Surfaces](7721-lic-on-stream-surfaces.md), by operating directly on the volume or 2D domain rather than requiring an intermediate surface geometry.

## How it works
**Inputs and Outputs:**
*   Input 1: A scalar field $N(x,y)$, typically white noise.
*   Input 2: A vector field $\vec{V}(x,y)$.
*   Output: A scalar field $L(x,y)$ representing the convolved texture.

**Data Structures:**
*   $N$: A 2D grid of scalar values (noise).
*   $\vec{V}$: A 2D grid of 2D vectors.
*   $L$: A 2D grid of scalar values (output).

**Steps:**
1.  **Initialize Noise:** Generate a white noise scalar field $N$. Each pixel value is drawn from a uniform distribution (e.g., $[0, 1]$ or $[-0.5, 0.5]$) [S1, unofficial].
2.  **Iterate Pixels:** For each pixel location $(x, y)$ in the output domain:
    a.  **Determine Direction:** Retrieve the vector $\vec{V}(x, y)$ at the current location [S1, unofficial].
    b.  **Rasterize Streamline:** Construct a line segment centered at $(x, y)$ oriented in the direction of $\vec{V}(x, y)$. The length of this segment is a parameter $L_{seg}$ [S1, unofficial].
    c.  **Sample Noise:** Identify all pixels in the noise field $N$ that intersect this rasterized line segment [S1, unofficial].
    d.  **Integrate:** Sum the noise values of the intersected pixels. This sum constitutes the LIC value for $(x, y)$ [S1, unofficial].
    e.  **Assign:** Store the sum in $L(x, y)$.

**Formula:**
The LIC value $L(x, y)$ at position $(x, y)$ is defined as:
$$ L(x, y) = \sum_{p \in \text{RasterizedLine}(x, y, \vec{V}(x, y))} N(p) $$
Where:
*   $\text{RasterizedLine}(x, y, \vec{V}(x, y))$ is the set of discrete pixel coordinates $p$ covered by the line segment centered at $(x, y)$ with direction $\vec{V}(x, y)$ [S1, unofficial].
*   $N(p)$ is the noise value at pixel $p$ [S1, unofficial].

**Parameters:**
*   Line segment length: Not explicitly defined in sources, but critical for smoothing. [G] Typically chosen to be several times the pixel size to ensure sufficient sampling.
*   Noise distribution: White noise [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Computational Cost:** The method is computationally expensive because it performs a separate integration for every pixel, leading to redundant calculations for adjacent pixels along the same streamline [S1, unofficial].
*   **Precision:** Rasterization artifacts may occur if the line segment is short or the vector field has high curvature, potentially blurring flow structure [S1, unofficial].
*   **Boundary Conditions:** The sources do not specify handling for vectors at domain boundaries. [G] Standard practice involves clamping or reflecting vectors, or limiting the line segment to stay within bounds.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def line_integral_convolution(noise_field, vector_field, line_length):
    """
    Computes Line Integral Convolution.
    
    Args:
        noise_field: 2D array of scalar noise values.
        vector_field: 2D array of 2D vectors (dx, dy).
        line_length: Length of the integration line segment.
        
    Returns:
        2D array of LIC values.
    """
    height, width = noise_field.shape
    lic_output = zeros_like(noise_field)
    
    for y in range(height):
        for x in range(width):
            # Get vector at current pixel
            vx, vy = vector_field[y, x]
            
            # Normalize vector to determine direction
            magnitude = sqrt(vx**2 + vy**2)
            if magnitude == 0:
                lic_output[y, x] = 0
                continue
            
            dir_x = vx / magnitude
            dir_y = vy / magnitude
            
            # Define line segment endpoints
            # Centered at (x, y) with length 'line_length'
            half_len = line_length / 2.0
            x1 = x - dir_x * half_len
            y1 = y - dir_y * half_len
            x2 = x + dir_x * half_len
            y2 = y + dir_y * half_len
            
            # Rasterize line and sum noise values
            # Note: Bresenham's line algorithm or similar rasterization is implied
            # by "rasterizing a line segment" [S1, unofficial].
            total_noise = 0.0
            count = 0
            
            # Iterate over pixels covered by the line segment
            # This is a conceptual implementation of rasterization
            for t in arange(0, 1.0, 0.01): # Step size depends on desired precision
                curr_x = x1 + (x2 - x1) * t
                curr_y = y1 + (y2 - y1) * t
                
                # Sample noise at nearest pixel
                # Bilinear interpolation could be used for smoother results [G]
                px = int(round(curr_x))
                py = int(round(curr_y))
                
                if 0 <= px < width and 0 <= py < height:
                    total_noise += noise_field[py, px]
                    count += 1
            
            if count > 0:
                lic_output[y, x] = total_noise / count # Average or sum? Source says "summing up" [S1, unofficial]
                # Correction: Source says "integrating (or summing up)". 
                # Usually LIC is normalized by length or count to maintain brightness.
                # However, strict adherence to "summing up" implies raw sum.
                # We will use raw sum as per source text "summing up the values" [S1, unofficial].
                lic_output[y, x] = total_noise
            else:
                lic_output[y, x] = 0
                
    return lic_output
```

## Subtopics

### [Seed Line Integral Convolution](8864-seed-line-integral-convolution.md)
Computes LIC only along streamlines from sparse seeds, reducing cost for sparse inputs but increasing aliasing risk.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of pixels and $L$ is the average number of pixels per line segment. It is computationally expensive compared to optimized versions that reuse calculations [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the noise field, vector field, and output.
*   **Strengths:** Effectively visualizes 2D vector fields by correlating noise along flow directions [S1, unofficial]. Provides a dense, continuous texture.
*   **Limitations:** High computational cost. Can blur flow structure in regions of high curvature or vorticity [S1, unofficial].
*   **When to use:** When a dense, texture-based visualization of 2D flow is required and computational resources are sufficient.
*   **When not to use:** For large-scale 3D volumes or real-time applications where performance is critical; consider [Seed Line Integral Convolution](8864-seed-line-integral-convolution.md) or [LIC on Stream Surfaces](7721-lic-on-stream-surfaces.md) instead.

## Background
LIC was introduced as one of the most visible recent flow visualization algorithms [S1, unofficial]. It operates on a discrete grid by rasterizing line segments and integrating noise values [S1, unofficial]. Stalling and Hege proposed extensions to speed up the process by reusing calculations for adjacent pixels along the same streamline [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
