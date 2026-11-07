# 3D Pixel Texture Classification

Also called: VGH Lookup

## Overview
3D Pixel Texture Classification is a hardware-accelerated volume rendering technique that maps multi-dimensional voxel attributes to color and opacity using a 3D texture lookup [S1]. It solves the problem of complex, multi-parameter classification by offloading the transfer function evaluation to the GPU's texture hardware, allowing for real-time interaction [S1, S2]. It differs from standard scalar ray casting by utilizing a 3D texture space defined by Data Value, Gradient Magnitude, and Second Derivative, rather than a 1D or 2D transfer function [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume $V(\mathbf{x})$.
*   **Output:** A 2D image with accumulated color and opacity.

**Data Structures:**
The core data structure is a 3D pixel texture (3D texture) used for classification [S1].
*   **Axis 1 (Data Value):** 8-bit resolution (256 entries) [S1].
*   **Axis 2 (Gradient Magnitude):** 8-bit resolution (256 entries) [S1].
*   **Axis 3 (Second Derivative):** Limited resolution [S1]. This axis represents "sheets" with specific control points [S1].

**Steps:**
1.  **Ray Casting:** Cast rays through the volume using the shared principles of [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md). For each sample point $\mathbf{p}_i$, retrieve the scalar value $V(\mathbf{p}_i)$ [S1, S2].
2.  **Attribute Computation:** Compute the gradient magnitude $|\nabla V|$ and the second derivative $V''$ at $\mathbf{p}_i$ [S1].
3.  **Second Derivative Preprocessing:** The second derivative is a signed quantity. Apply a scale and bias to maintain the sign information, ensuring proper interpolation within the 3D texture [S1].
4.  **Texture Lookup:** Use the computed attributes as coordinates to sample the 3D pixel texture [S1].
    *   The Data Value and Gradient Magnitude axes use full 8-bit resolution [S1].
    *   The Second Derivative axis uses limited control points representing "sheets" [S1].
5.  **Classification Logic:**
    *   The texture contains three control points for the second derivative axis: negative, zero, and positive [S1].
    *   Opacity on the center sheet (zero second derivative) is directly controlled by classification widgets [S1].
    *   Opacity on the outer sheets (positive and negative second derivatives) is scaled from the center sheet opacity based on a "boundary emphasis slider" [S1].
6.  **Compositing:** Use the retrieved color $C_i$ and opacity $\alpha_i$ in the front-to-back compositing equation [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume standard ray casting loop variables: ray_origin, ray_direction, step_size
# Assume volume V is accessible via sampler3D
# Assume classification texture T is a 3D texture with dimensions [256, 256, N_sec]

def classify_and_render(ray_origin, ray_direction):
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    
    # Initialize ray parameters
    t = 0.0
    max_t = distance_to_far_plane()
    
    while t < max_t and accumulated_alpha < 0.95: # Early ray termination
        # 1. Sample volume
        p = ray_origin + t * ray_direction
        scalar_val = texture3D(V, p).r
        
        # 2. Compute attributes
        # Gradient magnitude
        grad_mag = compute_gradient_magnitude(p)
        
        # Second derivative (signed)
        sec_deriv = compute_second_derivative(p)
        
        # 3. Preprocess second derivative for texture lookup
        # Scale and bias to maintain sign and map to texture coordinates
        sec_deriv_normalized = scale_and_bias(sec_deriv)
        
        # 4. Lookup in 3D classification texture
        # Coordinates: [Data Value, Gradient Magnitude, Second Derivative]
        # Data Value and Grad Mag are 8-bit (0-255)
        # Second Derivative uses limited control points
        color, alpha = texture3D(T, vec3(scalar_val, grad_mag, sec_deriv_normalized)).rgb, texture3D(T, vec3(scalar_val, grad_mag, sec_deriv_normalized)).a
        
        # 5. Composite
        # Front-to-back compositing
        accumulated_color = accumulated_color + (1.0 - accumulated_alpha) * alpha * color
        accumulated_alpha = accumulated_alpha + (1.0 - accumulated_alpha) * alpha
        
        t += step_size
    
    return accumulated_color, accumulated_alpha
```

**Parameters:**
*   **Data Value Resolution:** 8-bit (256 entries) [S1].
*   **Gradient Magnitude Resolution:** 8-bit (256 entries) [S1].
*   **Second Derivative Resolution:** Limited, using 3 control points (negative, zero, positive) [S1].
*   **Boundary Emphasis Slider:** Controls the scaling of opacity for outer sheets relative to the center sheet [S1].

**Edge Cases and Pitfalls:**
*   **Sign of Second Derivative:** Must be maintained via scale and bias to allow proper interpolation [S1].
*   **Texture Size:** Full 8-bit resolution for all three axes would be prohibitively large [S1].

## Subtopics

### Adaptive Pre-integrated Voxel Classification
A technique included in a real-time raycaster built by Roettger et al. to exploit the intrinsically parallel nature of GPUs, along with volume clipping [S2]. It addresses the inefficiency of multi-pass rendering pipelines by using a single-pass framework [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The 3D classification texture size is reduced by limiting the resolution of the second derivative axis to avoid prohibitive memory usage [S1].
*   **Performance:** Enables real-time raycasting when combined with GPU parallelism and volume clipping [S2].
*   **Strengths:** Allows complex multi-dimensional classification (Data Value, Gradient, Second Derivative) to be evaluated efficiently via hardware texture lookup [S1].
*   **Limitations:** The size of the 3D texture is constrained; full 8-bit resolution for all axes is not feasible [S1].

## Background
The method was introduced in the context of interactive volume rendering using multi-dimensional transfer functions [S1]. It was implemented as part of a system that utilized 3D pixel textures for classification [S1]. Later, similar concepts were integrated into real-time GPU-based raycasters by Roettger et al., who combined adaptive pre-integrated voxel classification with volume clipping to improve performance over multi-pass approaches [S2].

## Sources
[S1] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
