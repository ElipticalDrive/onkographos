# 2D Gaussian Evaluation (Rotated/Anisotropic)
Also called: 2D Gaussian Evaluation with Rotation, Brush stroke texture generation via Gaussian combination

## Overview
This method computes the alpha component of a rotated, anisotropic 2D Gaussian function at specific 2D texture coordinates, primarily for generating brush-stroke textures in stroke-based splatting [S1]. It differs from standard isotropic splatting by applying a rotation matrix to the coordinate system before evaluating the decay, allowing for directional brush strokes [S1]. It builds on [2D Gaussian Evaluation](4728-2d-gaussian-evaluation.md) by adding explicit rotational transformation of the input coordinates [G].

## How it works
**Inputs:**
- $P = (u, v)$: The 2D texture coordinates of the fragment being evaluated [S1].
- $A$: Amplitude of the Gaussian function, controlling overall opacity [S1].
- $P_0 = (x_0, y_0)$: The center of the Gaussian function in texture space coordinates [S1].
- $\sigma_x, \sigma_y$: Gaussian function decay parameters along the local x and y axes [S1].
- $\lambda$: Gaussian shape exponent controlling the roundness of the function [S1].
- $\theta$: Orientation of the function in texture space coordinates [S1].

**Outputs:**
- $z$: The resulting alpha value in RGBA space [S1].

**Data Structures:**
No complex data structures are required for the evaluation itself [S1]. The parameters ($A, P_0, \sigma_x, \sigma_y, \theta$) are typically passed as uniforms or varyings from the geometry shader to the fragment shader [S1].

**Steps:**
1. Calculate the displacement vector from the center: $\Delta u = u - x_0$, $\Delta v = v - y_0$ [S1].
2. Rotate the displacement vector by angle $\theta$ to obtain local coordinates $(x_\theta, y_\theta)$: $x_\theta = \cos \theta \cdot \Delta u - \sin \theta \cdot \Delta v + x_0$, $y_\theta = \sin \theta \cdot \Delta u + \cos \theta \cdot \Delta v + y_0$ [S1]. This effectively rotates the point around the origin and then translates, or rotates the vector and re-centers [G]. Standard rotation of a vector $(\Delta u, \Delta v)$ yields $(\cos \theta \Delta u - \sin \theta \Delta v, \sin \theta \Delta u + \cos \theta \Delta v)$ [G]. The source adds $x_0$ to the x-component and $y_0$ to the y-component [S1]. This implies the formula evaluates the Gaussian at a point that has been rotated relative to the center, but the final coordinates used in the decay term are $x_\theta$ and $y_\theta$ as defined [S1].
3. Evaluate the Gaussian decay using the rotated coordinates: $z = A \cdot e^{-\frac{1}{2} \left( \frac{x_\theta - x_0}{\sigma_x} \right)^2} \cdot e^{-\frac{1}{2} \left( \frac{y_\theta - y_0}{\sigma_y} \right)^2}$ [S1]. The source text contains OCR errors (`greenx`, `greey`) for $x_\theta$ and $y_\theta$ [S1]. The denominator in the source snippet shows $\sigma_x$ for both terms in the garbled text, but the text description explicitly states $\sigma_x$ and $\sigma_y$ are decay along x and y axes respectively [S1]. We assume anisotropic decay $\sigma_x$ and $\sigma_y$ [S1].
4. Return $z$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_rotated_gaussian(u, v, A, x0, y0, sigma_x, sigma_y, theta):
    # Step 1: Displacement
    delta_u = u - x0
    delta_v = v - y0
    
    # Step 2: Rotation
    cos_theta = cos(theta)
    sin_theta = sin(theta)
    
    # The source formula adds x0/y0 back to the rotated components.
    # Mathematically, (x_theta - x0) simplifies to the rotated displacement.
    rot_u = cos_theta * delta_u - sin_theta * delta_v
    rot_v = sin_theta * delta_u + cos_theta * delta_v
    
    # Step 3: Gaussian Evaluation
    # Using anisotropic sigma as described in text
    exponent_x = -0.5 * (rot_u / sigma_x) ** 2
    exponent_y = -0.5 * (rot_v / sigma_y) ** 2
    
    z = A * exp(exponent_x) * exp(exponent_y)
    
    return z
```

**Parameters:**
- $A$: User-defined, controls opacity [S1].
- $\sigma_x, \sigma_y$: User-defined, controls spread [S1]. Can be adjusted based on cluster shape (e.g., longer brushes for longer clusters) [S1].
- $\theta$: Modified by a noise function based on point coordinates and normals to introduce variation between neighboring patches [S1].
- $\lambda$: Shape exponent [S1]. Source [S1] lists it as a parameter but the provided formula snippet does not explicitly contain $\lambda$ [S1]. It is likely used to modify the exponent power or scale the sigmas, but the exact integration into the formula is not given in the snippet [G].
- Grid Disposition: Typically 6 to 9 Gaussians on a 3x3 grid are used to simulate brush styles [S1]. Separation is controlled by `dev` [S1].

**Edge Cases and Pitfalls:**
- **Precision**: Standard floating-point precision is sufficient for the exponential decay [G].
- **Boundaries**: The Gaussian extends infinitely; in practice, values below a threshold (e.g., 0.01) can be clamped to 0 for performance [G].
- **Degenerate Input**: If $\sigma_x$ or $\sigma_y$ is 0, the function is undefined (division by zero) [G]. Implementation should clamp sigmas to a minimum epsilon [G].

## Subtopics

### Generalized Gaussian Particles
Uses a shape exponent $\lambda$ to control the tail behavior of the Gaussian [G]. Choose when non-Gaussian tails are needed for artistic control [G]. See [Generalized Gaussian Particles](9164-generalized-gaussian-particles.md).

## Operating characteristics
- **Time Complexity**: $O(1)$ per Gaussian evaluation [G]. Constant number of trigonometric and exponential operations [G].
- **Memory Complexity**: $O(1)$ per fragment [G]. Parameters are passed via varyings [S1].
- **Performance**: Implemented in real-time using geometry and fragment shaders [S1]. Performance impact is negligible even when doubling the number of Gaussians for two-layer painting [S1].
- **Strengths**: Efficiently generates directional, anisotropic brush strokes [S1]. Simple to implement in shaders [G].
- **Limitations**: The formula provided is specific to the brush-stroke context [S1]. The role of $\lambda$ is not explicitly defined in the formula [S1].

## Background
Originated in the context of stroke-based splatting for point cloud visualization and artistic rendering [S1]. The method was introduced by R. K. dos Anjos et al. in "Stroke-based splatting: an efficient multi-resolution point cloud visualization technique" [S1]. It is used to simulate brush textures by combining multiple rotated Gaussian functions, where the resulting alpha value represents the transparency of the brush stroke [S1].

## Sources
[S1] Stroke-based splatting: an efficient multi-resolution point cloud visualization technique, https://doi.org/10.1007/s00371-017-1420-7, fetched 2026-10-26
