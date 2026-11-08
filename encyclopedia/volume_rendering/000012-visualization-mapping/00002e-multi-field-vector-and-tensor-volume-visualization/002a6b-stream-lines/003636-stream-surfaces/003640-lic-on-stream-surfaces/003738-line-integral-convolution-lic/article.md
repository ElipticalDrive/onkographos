# Line Integral Convolution (LIC)
## Overview
Line Integral Convolution (LIC) is a texture-based visualization method that maps a vector field by convolving a scalar noise field along the field's streamlines [S1, unofficial]. It solves the problem of visualizing flow direction and structure in 2D or 3D domains by generating an output scalar field where noise values are correlated with the local flow direction [S1, unofficial]. Unlike direct streamline plotting, LIC provides a continuous, dense texture that reveals global flow patterns, though it is computationally more expensive than optimized variants like Fast LIC [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input 1: A scalar field, typically a white noise image $N(x,y)$ [S1, unofficial].
- Input 2: A vector field $\vec{V}(x,y)$ defined on a discrete grid [S1, unofficial].
- Output: A scalar field $L(x,y)$ representing the convolved noise [S1, unofficial].

**Data Structures:**
- The vector field is stored as a grid of 2D vectors $\vec{V}_{i,j}$ corresponding to pixel coordinates $(i,j)$ [S1, unofficial].
- The noise field is a grid of scalar values $N_{i,j}$ [S1, unofficial].
- The output grid $L_{i,j}$ has the same dimensions as the input fields [S1, unofficial].

**Steps:**
1. For each pixel $(x,y)$ in the output domain:
2. Construct a line segment (streamline approximation) centered at $(x,y)$ and oriented in the direction of $\vec{V}(x,y)$ [S1, unofficial].
3. Rasterize this line segment to identify the set of pixels $\mathcal{P}_{x,y}$ that the line passes through [S1, unofficial].
4. Compute the LIC value $L(x,y)$ by summing the noise values at the rasterized pixels:
   $$ L(x,y) = \sum_{(u,v) \in \mathcal{P}_{x,y}} N(u,v) $$
   [S1, unofficial]
5. Repeat for all pixels $(x,y)$ in the domain [S1, unofficial].

**Parameters:**
- Line segment length: Not specified in the sources. [G] Typically determined by the integration step size and number of steps in streamline tracing.
- Noise field: White noise, meaning values are uncorrelated and uniformly or normally distributed. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Computational Cost:** LIC is computationally expensive because it requires independent integration for each pixel, leading to redundant calculations for adjacent pixels along the same streamline [S1, unofficial].
- **Precision:** The method operates on a discrete grid; rasterization errors may affect the accuracy of the integral [S1, unofficial].
- **Boundaries:** The sources do not specify boundary handling for streamlines exiting the domain. [G] Standard practice involves clamping or stopping integration at domain boundaries.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def line_integral_convolution(noise_field, vector_field, output_shape):
    """
    Computes Line Integral Convolution for a 2D vector field.
    
    Args:
        noise_field: 2D array of scalar noise values.
        vector_field: 2D array of 2D vectors (dx, dy).
        output_shape: Tuple (height, width) of the output grid.
        
    Returns:
        2D array of convolved scalar values.
    """
    height, width = output_shape
    output = zeros(output_shape)
    
    for y in range(height):
        for x in range(width):
            # Get vector at current pixel
            vx, vy = vector_field[y, x]
            
            # Normalize vector to get direction
            magnitude = sqrt(vx**2 + vy**2)
            if magnitude == 0:
                output[y, x] = 0
                continue
            
            dir_x = vx / magnitude
            dir_y = vy / magnitude
            
            # Define line segment centered at (x, y)
            # Note: Length parameter is not specified in sources [G]
            # Assuming a fixed length L for the segment
            L = 10.0 # Placeholder for line length, not given in sources
            
            # Rasterize line segment to get set of pixels
            # This step involves determining which grid cells the line intersects
            # Standard Bresenham or DDA algorithm can be used
            pixels_on_line = rasterize_line(x, y, dir_x, dir_y, L)
            
            # Sum noise values for pixels on the line
            total_noise = 0.0
            for (px, py) in pixels_on_line:
                # Check bounds
                if 0 <= px < width and 0 <= py < height:
                    total_noise += noise_field[py, px]
            
            output[y, x] = total_noise
            
    return output

def rasterize_line(start_x, start_y, dir_x, dir_y, length):
    """
    Rasterizes a line segment starting at (start_x, start_y) with direction (dir_x, dir_y)
    and given length. Returns a list of pixel coordinates.
    """
    # Implementation details for rasterization are not provided in sources [G]
    # This is a standard computer graphics operation
    pass
```
## Subtopics

### [Seed Line Integral Convolution](8864-seed-line-integral-convolution.md)
Computes LIC only along streamlines from sparse seed points, reducing computational cost for sparse inputs but increasing susceptibility to aliasing.

### [Image Space Fabric Visualization](10569-image-space-fabric-visualization.md)
Projects tensor data into image space and adapts LIC advection for real-time GPU rendering of symmetric tensor fields on surfaces.

## Operating characteristics
- **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of pixels and $L$ is the average number of pixels per streamline segment. The method is computationally expensive due to independent integration per pixel [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing the input noise, vector, and output fields [S1, unofficial].
- **Performance:** LIC is significantly slower than optimized versions like Fast LIC, which reuse calculations for adjacent pixels along the same streamline [S1, unofficial].
- **Strengths:** Effective for visualizing 2D vector fields with continuous texture [S1, unofficial].
- **Limitations:** High computational cost; not suitable for real-time applications without optimization [S1, unofficial].
- **When to use:** When dense, continuous flow visualization is required and computational resources are sufficient.
- **When not to use:** For large-scale or real-time applications where performance is critical; consider Fast LIC or Seed LIC instead.
## Background
Line Integral Convolution was introduced as one of the most visible recent flow visualization algorithms [S1, unofficial]. It was proposed by Cabral and Leedom in 1991 [S1, inferred from citation [18, unofficial] in source text, though exact reference details are not provided in the source text]. The method has been widely used for visualizing 2D and 3D vector fields in scientific visualization.
## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
