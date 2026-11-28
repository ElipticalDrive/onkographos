# Contour Rendering

Also called: Contour Lines Rendering, Gradient-based Contouring, Non-Photorealistic Volume Rendering for Contours, Object Contour Visualization

## Overview
Contour Rendering is a non-photorealistic volume rendering technique that visualizes object boundaries by leveraging gradient magnitude and viewing angle, rather than direct data values or transfer functions [S1][S2, unofficial]. It solves the problem of visual clutter in volumetric data by producing sparse images containing only significant contours, avoiding the need for complex iso-surface extraction or transfer function tuning [S1]. Unlike [Line Illustration Detection](772-line-illustration-detection.md) which focuses on flow or texture lines, or [Outline and Contour Generation](4137-outline-and-contour-generation.md) which often relies on geometric primitives, Contour Rendering operates directly on volumetric gradient fields to highlight internal structures [S1][S2, unofficial].

## How it works
**Inputs:** Volumetric data $D$, viewing direction $V$, gradient threshold $t_g$, exponent $n$, and windowing parameters [S1][S2, unofficial].
**Outputs:** A 2D image where pixels represent contour intensities [S1].

**Data Structures:**
*   **Gradient Field:** A 3D grid of vectors $\nabla D$, computed via 4D linear regression for high quality [S1][S2, unofficial].
*   **Windowing Function $g(|\nabla|)$:** A scalar field mapping gradient magnitudes to "surfaceness" values [S1].
*   **Sample Points:** Points along viewing rays with computed intensity $I(P, V)$ [S1].

**Steps:**

1.  **Gradient Estimation:** Compute the gradient magnitude $|\nabla(P)|$ for each voxel $P$ in the volume [S1]. High-quality reconstruction is achieved using 4D linear regression [S1][S2, unofficial].
2.  **Thresholding:** For each ray sample at position $P$, check if $|\nabla(P)| > t_g$ [S2, unofficial]. If not, discard the sample [S2, unofficial].
3.  **Surfaceness Calculation:** Apply the windowing function $g(|\nabla(P)|)$ to the gradient magnitude [S1]. This function characterizes the "surfaceness" of the voxel, emphasizing regions with significant data value changes [S1].
4.  **View-Dependent Weighting:** Calculate the view-dependent weight $s(P, V)$ using the formula:
    $$s(P, V) = (1 - |\nabla(P) \cdot V|)^n$$
    where $V$ is the normalized viewing direction vector and $n$ is an exponent controlling contour sharpness [S1].
5.  **Intensity Calculation:** Compute the view-dependent intensity $I(P, V)$ for valid samples:
    $$I(P, V) = g(|\nabla(P)|) \cdot s(P, V)$$
    This results in a sparse volume where only voxels with significant contours contribute [S1].
6.  **Compositing:** Choose one of the following compositing strategies [S1]:
    *   **Maximum Intensity Projection (MIP):** Assign the highest sample intensity $I(P, V)$ along the ray to the pixel [S1].
    *   **Depth-Cued MIP:** Weight sample intensities by distance from the viewpoint to improve spatial impression [S1].
    *   **Local Maximum Intensity Projection (LMIP):** Project the first local maximum intensity along the ray that exceeds a predefined threshold, preventing higher intensity contours from hiding weaker ones closer to the viewer [S1].
    *   **Alpha-Blending:** Assign $g(|\nabla(P)|)$ as color and opacity, modulated by $I(P, V)$, and perform alpha-blending [S1]. This emphasizes low-curvature surfaces and is suitable for data with many high-frequency regions [S1].
7.  **Optional Extension (Level Lines):** Select specific slices (e.g., every twentieth slice) [S1]. Assign values of $g$ as intensities to these slices and render them using MIP with a different color channel to distinguish from object contours [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def contour_rendering(volume_shape, gradient_field, view_direction, t_g, n, compositing_mode='MIP'):
    """
    Render contours from volumetric data using gradient magnitude and view direction.
    
    Args:
        volume_shape: Tuple (height, width, depth) of the volume grid.
        gradient_field: 3D list of gradient vectors (computed via 4D linear regression).
        view_direction: Normalized 3D list representing the camera view direction.
        t_g: Gradient magnitude threshold.
        n: Exponent for view-dependent weighting.
        compositing_mode: 'MIP', 'LMIP', or 'AlphaBlend'.
        
    Returns:
        2D list of contour intensities.
    """
    height, width, depth = volume_shape
    image = [[0.0] * width for _ in range(height)]
    
    # Precompute gradient magnitudes
    grad_mag = [[[math.sqrt(sum(c*c for c in grad)) for grad in row] for row in slice] for slice in gradient_field]
    
    for y in range(height):
        for x in range(width):
            max_intensity = -1.0
            local_max_intensity = -1.0
            alpha_accumulated = 0.0
            color_accumulated = 0.0
            
            # Ray traversal along depth axis
            for z in range(depth):
                # Step 2: Thresholding
                g = grad_mag[y][x][z]
                if g <= t_g:
                    continue
                
                # Step 3: Surfaceness
                g_val = windowing_function(g)
                
                # Step 4: View-dependent weight
                grad_vec = gradient_field[y][x][z]
                grad_len = math.sqrt(sum(c*c for c in grad_vec))
                if grad_len == 0:
                    continue
                grad_dir = [c / grad_len for c in grad_vec]
                
                dot = abs(sum(g * v for g, v in zip(grad_dir, view_direction)))
                s_val = (1.0 - dot) ** n
                
                # Step 5: Intensity
                intensity = g_val * s_val
                
                # Step 6: Compositing
                if compositing_mode == 'MIP':
                    if intensity > max_intensity:
                        max_intensity = intensity
                elif compositing_mode == 'LMIP':
                    if intensity > local_max_intensity:
                        local_max_intensity = intensity
                elif compositing_mode == 'AlphaBlend':
                    alpha = g_val
                    color = g_val
                    color_accumulated = color_accumulated * (1.0 - alpha) + color * alpha
                    alpha_accumulated += alpha * (1.0 - alpha_accumulated)
            
            if compositing_mode == 'MIP':
                image[y][x] = max_intensity
            elif compositing_mode == 'LMIP':
                image[y][x] = local_max_intensity
            elif compositing_mode == 'AlphaBlend':
                image[y][x] = color_accumulated
    return image

def windowing_function(grad_mag):
    # Binary threshold implementation [S2, unofficial]
    # Note: [S1] describes a smooth windowing function for 'surfaceness', 
    # but exact shape is not specified. Returns 1.0 for significant gradients.
    return 1.0 if grad_mag > 0 else 0.0
```

**Parameters:**
*   $t_g$: Gradient magnitude threshold. Determines which samples are considered contours [S2, unofficial].
*   $n$: Exponent controlling the sharpness of the contour in the view-dependent function [S1].
*   Windowing parameters: Define the range of gradient magnitudes of interest [S1].

**Edge Cases and Pitfalls:**
*   **Gradient Estimation:** Poor gradient estimation leads to poor results. 4D linear regression is recommended over central differences [S1][S2, unofficial].
*   **Depth Information:** Simple MIP does not provide depth cues. Use depth cueing or LMIP to mitigate [S1].
*   **Occlusion:** In MIP, higher intensity contours can hide weaker contours closer to the viewer. LMIP handles this by projecting the first local maximum above a threshold [S1].
*   **High-Frequency Data:** Alpha-blending may hide fine details in data with many high-frequency regions, as it emphasizes low-curvature surfaces [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels along the ray, but significantly reduced due to sparsity (only voxels with significant gradients contribute) [S1].
*   **Memory Complexity:** Requires storage for the volume and gradient field [S1].
*   **Performance:** Interactive frame rates can be achieved without specialized hardware due to the sparse nature of the contribution [S1].
*   **Strengths:** No need for transfer function tuning; highlights internal structures without visual overload; view-dependent [S1][S2, unofficial].
*   **Limitations:** No direct depth information in basic MIP; sensitive to gradient estimation quality; may hide fine details in high-frequency data with alpha-blending [S1].

## Background
Contour Rendering was introduced as a non-photorealistic rendering technique for volumetric data [S1]. It was developed to visualize 3D structures characterized by significant changes in data values without being obstructed by continuous regions [S1]. The method relies on high-quality gradient reconstruction, specifically using 4D linear regression [S1][S2, unofficial]. It is used in medical imaging and other fields where clear visualization of object boundaries is required without the complexity of iso-surface extraction [S1][S2, unofficial].

## Sources
[S1] Fast Visualization of Object Contours by Non-Photorealistic Volume Rendering, https://doi.org/10.1111/1467-8659.00538, fetched 2026-11-01
[S2] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
