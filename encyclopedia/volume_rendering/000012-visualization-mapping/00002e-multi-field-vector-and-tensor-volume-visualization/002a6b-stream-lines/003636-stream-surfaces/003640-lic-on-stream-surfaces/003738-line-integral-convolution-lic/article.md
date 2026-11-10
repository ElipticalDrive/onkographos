# Line Integral Convolution (LIC)
Also called: 线积分卷积
## Overview
Line Integral Convolution (LIC) is a texture-based flow visualization method that densely displays direction information across an entire vector field without requiring seed points [S2, unofficial]. It solves the problem of sparse or discontinuous flow representation by convolving a random scalar noise field along the streamlines of the input vector field, producing streak-like textures that intuitively show flow direction [S1, S2, unofficial]. Unlike streamline methods that rely on discrete seed points, LIC provides a continuous, global overview of the flow structure [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input 1:** A 2D vector field $\mathbf{V}(x, y)$ defined on a discrete grid.
*   **Input 2:** A scalar noise field $N(x, y)$, typically white noise [S1, S2, unofficial].
*   **Output:** A scalar field $L(x, y)$ representing the convolved texture [S1, unofficial].

**Data Structures:**
*   The vector field is stored as a 2D array of 2D vectors.
*   The noise field is stored as a 2D array of scalar values.
*   The output is a 2D array of scalar values with the same dimensions as the input grid.

**Algorithm Steps:**
1.  **Generate Noise:** Create a white noise image $N(x, y)$ where pixel values are random [S1, S2, unofficial].
2.  **Rasterize Streamlines:** For each pixel $(x, y)$ in the output grid, determine the local vector $\mathbf{V}(x, y)$. Rasterize a line segment centered at $(x, y)$ oriented in the direction of $\mathbf{V}(x, y)$ [S1, unofficial].
3.  **Integrate:** Sum the values of the noise field $N$ at the pixels covered by the rasterized line segment [S1, unofficial].
4.  **Assign:** Assign the sum to the output pixel $L(x, y)$ [S1, unofficial].

**Formula:**
The LIC value $L(x, y)$ at a point $(x, y)$ is computed as:
$$ L(x, y) = \int_{-L/2}^{L/2} N(\mathbf{r}(t)) \, dt $$
where $\mathbf{r}(t)$ is the streamline passing through $(x, y)$, and $L$ is the integration length [S1, S2, unofficial]. In discrete implementation, this integral is approximated by summing noise values along the rasterized line segment [S1, unofficial].

**Parameters:**
*   **Integration Length:** The length of the line segment used for convolution. Not given in the sources.
*   **Noise Type:** Typically white noise [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Direction Ambiguity:** LIC does not distinguish between positive and negative flow directions; the texture looks the same for opposite vectors [S2, unofficial].
*   **Speed Magnitude:** LIC does not display flow speed magnitude [S2, unofficial].
*   **Computational Cost:** Standard LIC is computationally expensive because it performs independent integrations for every pixel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def line_integral_convolution(vector_field, noise_field, integration_length):
    """
    Computes Line Integral Convolution for a 2D vector field.
    
    Args:
        vector_field: 2D array of (vx, vy) vectors.
        noise_field: 2D array of scalar noise values.
        integration_length: Length of the integration segment.
        
    Returns:
        2D array of convolved scalar values.
    """
    height, width = vector_field.shape
    output = zeros_like(noise_field)
    
    # Precompute streamline paths for efficiency [S1, unofficial]
    # Note: Standard LIC recomputes or approximates for each pixel [S1, unofficial]
    
    for y in range(height):
        for x in range(width):
            # Get local vector
            vx, vy = vector_field[y, x]
            
            # Normalize vector to get direction
            magnitude = sqrt(vx**2 + vy**2)
            if magnitude == 0:
                output[y, x] = noise_field[y, x]
                continue
                
            dir_x = vx / magnitude
            dir_y = vy / magnitude
            
            # Rasterize line segment centered at (x, y)
            # Approximate integration by sampling points along the segment
            sum_noise = 0.0
            num_samples = int(integration_length)
            
            for i in range(-num_samples // 2, num_samples // 2 + 1):
                # Calculate position along streamline
                t = i * (integration_length / num_samples)
                sample_x = x + dir_x * t
                sample_y = y + dir_y * t
                
                # Sample noise field (interpolation required for non-integer coords)
                # [G] Bilinear interpolation is standard for sampling noise
                noise_val = sample_noise(noise_field, sample_x, sample_y)
                sum_noise += noise_val
                
            output[y, x] = sum_noise
            
    return output

def sample_noise(noise_field, x, y):
    """
    Samples noise field at non-integer coordinates using bilinear interpolation.
    """
    x0 = floor(x)
    y0 = floor(y)
    x1 = x0 + 1
    y1 = y0 + 1
    
    # Handle boundaries [G]
    x0 = clamp(x0, 0, noise_field.shape[1] - 1)
    y0 = clamp(y0, 0, noise_field.shape[0] - 1)
    x1 = clamp(x1, 0, noise_field.shape[1] - 1)
    y1 = clamp(y1, 0, noise_field.shape[0] - 1)
    
    # Bilinear interpolation weights
    dx = x - x0
    dy = y - y0
    
    val00 = noise_field[y0, x0]
    val10 = noise_field[y0, x1]
    val01 = noise_field[y1, x0]
    val11 = noise_field[y1, x1]
    
    top = val00 * (1 - dx) + val10 * dx
    bottom = val01 * (1 - dx) + val11 * dx
    
    return top * (1 - dy) + bottom * dy
```

## Subtopics

### [Seed Line Integral Convolution](8864-seed-line-integral-convolution.md)
Computes LIC only along field lines from sparse seed points, reducing computational cost for sparse inputs but increasing susceptibility to aliasing.

### [Image Space Fabric Visualization](10569-image-space-fabric-visualization.md)
Adapts LIC to visualize symmetric tensor fields on surfaces by projecting tensor data into image space and advecting noise along eigenvectors, enabling real-time performance on GPU.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of pixels and $L$ is the integration length. Standard LIC is computationally expensive [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the input fields and output texture.
*   **Strengths:** Densely displays direction information across the entire field without seed points [S2, unofficial]. Intuitively shows flow direction via streak-like textures [S2, unofficial].
*   **Limitations:** Does not display flow direction (positive/negative looks the same) [S2, unofficial]. Does not show flow speed magnitude [S2, unofficial]. Computationally expensive compared to optimized versions [S1, unofficial].
*   **When to Use:** When a global, continuous overview of flow direction is needed without selecting seed points [S2, unofficial].
*   **When Not to Use:** When flow direction (sign) or speed magnitude is critical, or when computational resources are limited [S2, unofficial].

## Background
Line Integral Convolution was introduced as one of the most visible recent flow visualization algorithms [S1, unofficial]. It operates on a discrete grid by rasterizing line segments and integrating noise values [S1, unofficial]. Stalling and Hege proposed an extension to speed up the process by reusing calculations for adjacent pixels along the same streamline [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] 第9 章：科学可视化, http://simingchen.me/smart-textbook/ch9/index.html, fetched 2026-10-30 (unofficial)
