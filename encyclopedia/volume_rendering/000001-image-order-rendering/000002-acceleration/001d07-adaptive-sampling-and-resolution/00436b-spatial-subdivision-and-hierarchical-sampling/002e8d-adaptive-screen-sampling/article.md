# Adaptive Screen Sampling

Also called: Gradient-based Adaptive Sampling

## Overview
Adaptive screen sampling is a spatially adaptive rendering technique that adjusts the number of rays cast per pixel based on the local color or intensity gradient in the image plane [S1, S2, unofficial]. It solves the problem of inefficient uniform sampling by concentrating computational effort in regions of high visual complexity (high gradients) while maintaining coarse sampling in smooth areas [S1, S2, unofficial]. Unlike hierarchical methods that use explicit data structures like quadtrees, this method operates directly on the 2D image buffer, switching to a higher sampling rate only where the color change in a small neighborhood exceeds a threshold [S1, S2, unofficial].

## How it works
**Inputs:**
*   Initial render buffer containing color/intensity values from a base sampling pass (typically 1 ray per pixel) [S1, S2, unofficial].
*   A threshold parameter $\tau$ for gradient magnitude [G].
*   A neighborhood size $N \times N$ (typically small, e.g., $3 \times 3$ or $5 \times 5$) [G].

**Outputs:**
*   Final image buffer with refined samples in high-gradient regions [S1, S2, unofficial].

**Data Structures:**
*   `ImageBuffer`: 2D array of color values $C(x,y)$ and/or intensity values $I(x,y)$ [S1, S2, unofficial].
*   `GradientMap`: 2D array of gradient magnitudes $G(x,y)$ [G].
*   `Mask`: 2D boolean array indicating pixels requiring refinement [G].

**Steps:**
1.  **Initial Rendering:** Cast a base set of rays (e.g., 1 per pixel) through the volume to generate the initial image buffer [S1, S2, unofficial].
2.  **Gradient Estimation:** Compute the gradient magnitude for each pixel in the initial buffer. For a pixel at $(x,y)$, the gradient magnitude $G(x,y)$ is calculated using the color or intensity values of the pixel and its neighbors in the neighborhood $N$ [S1, S2, unofficial].
    $$ G(x,y) = \sqrt{ \left( \frac{\partial C}{\partial x} \right)^2 + \left( \frac{\partial C}{\partial y} \right)^2 } $$
    where $\frac{\partial C}{\partial x}$ and $\frac{\partial C}{\partial y}$ are finite differences of the color/intensity channel [G].
3.  **Mask Generation:** Compare $G(x,y)$ against the threshold $\tau$. If $G(x,y) > \tau$, mark the pixel for refinement [S1, S2, unofficial].
4.  **Adaptive Refinement:** For each marked pixel, cast additional rays at a higher sampling rate to replace the interpolated or initial values with exact color values [S1, S2, unofficial].
5.  **Handling Misses:** If rays miss the volume surface during refinement, interpolate the missing values from surrounding valid samples [S1, S2, unofficial].
6.  **Reconstruction:** Combine the refined samples from marked pixels with the initial samples from unmarked pixels to form the final image [S2, S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_screen_sampling(initial_buffer, threshold, neighborhood_size):
    """
    initial_buffer: 2D array of color/intensity values from base pass
    threshold: float, gradient magnitude threshold
    neighborhood_size: int, size of the neighborhood for gradient calculation
    """
    height, width = initial_buffer.shape
    gradient_map = zeros((height, width))
    mask = zeros((height, width), dtype=bool)
    
    # Step 2: Gradient Estimation
    for y in range(height):
        for x in range(width):
            # Calculate gradient magnitude using finite differences
            # Handle boundaries by clamping or assuming zero gradient
            dx = initial_buffer[y, min(x+1, width-1)] - initial_buffer[y, max(x-1, 0)]
            dy = initial_buffer[min(y+1, height-1), x] - initial_buffer[max(y-1, 0), x]
            gradient_map[y, x] = sqrt(dx**2 + dy**2)
            
            # Step 3: Mask Generation
            if gradient_map[y, x] > threshold:
                mask[y, x] = True
                
    # Step 4 & 5: Adaptive Refinement
    final_buffer = copy(initial_buffer)
    for y in range(height):
        for x in range(width):
            if mask[y, x]:
                # Cast additional rays at higher sampling rate
                refined_color = cast_additional_rays(x, y, sampling_rate=4) # [G]
                
                # Handle misses: if refined_color is invalid, interpolate
                if refined_color is None:
                    refined_color = interpolate_neighbors(final_buffer, x, y)
                    
                final_buffer[y, x] = refined_color
                
    return final_buffer
```

**Parameters:**
*   $\tau$: Gradient threshold. Not given in the sources. [G]
*   Neighborhood size: Typically small (e.g., $3 \times 3$). Not given in the sources. [G]

**Edge Cases and Pitfalls:**
*   **Ray Misses:** Rays may occasionally miss the surface during refinement, requiring interpolation of missing values [S1, S2, unofficial].
*   **Boundary Conditions:** Gradient calculation at image boundaries requires special handling (e.g., clamping, mirroring, or assuming zero gradient) [G].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the average number of additional rays cast per pixel in high-gradient regions. In smooth regions, $M=1$; in complex regions, $M > 1$ [S1, S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the initial buffer, gradient map, and mask [G].
*   **Strengths:** Simple to implement; adapts to visual complexity without complex data structures; effective for scenes with sharp edges or high-frequency details [S1, S2, unofficial].
*   **Limitations:** Does not account for temporal coherence (if used in animation); may produce visible artifacts if the threshold is not chosen carefully; interpolation for missed rays can introduce blurring [S1, S2, unofficial].
*   **When to use:** When rendering static scenes with varying levels of detail across the image plane and computational resources are limited [S1, S2, unofficial].
*   **When not to use:** When temporal consistency is critical (e.g., real-time animation) or when the scene has uniform complexity [S1, S2, unofficial].

## Background
Adaptive screen sampling was introduced by Levoy in 1990 [S1, S2, unofficial]. It is well known in traditional ray tracing and has been applied to volume rendering to optimize the trade-off between rendering quality and performance [S1, S2, unofficial]. The method is particularly useful in applications where rendering speed is important, and the scene contains regions of both high and low visual complexity [S1, S2, unofficial].

## Sources
[S1] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S2] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
