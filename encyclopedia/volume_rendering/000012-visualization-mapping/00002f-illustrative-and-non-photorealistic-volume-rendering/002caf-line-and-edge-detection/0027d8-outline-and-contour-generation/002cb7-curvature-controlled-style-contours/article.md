# Curvature-Controlled Style Contours

Also called: Kindlmann Contours, Normal Curvature Contours

## Overview
Curvature-Controlled Style Contours is a non-photorealistic volume rendering technique that generates outline contours with consistent visual thickness by regulating the detection threshold based on the normal curvature along the view direction [S1]. Unlike standard dot-product contours, which produce uncontrolled thickness variations (too thick on flat surfaces, too thin on fine structures), this method ensures that contours remain perceptually uniform across varying surface geometries [S1]. It builds on [Outline and Contour Generation](4137-outline-and-contour-generation.md) by integrating curvature-aware edge detection with style transfer functions for color mapping [G].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric dataset, view vector $\mathbf{v}$, user-defined thickness parameter $T$, and a style transfer function (e.g., lit sphere map) [S1].
*   **Output:** Rendered volume image with stylized contours [S1].

**Data Structures:**
*   **Volume Data:** 3D texture or grid storing scalar values and normals $\mathbf{n}$ [S1].
*   **Ray Casting State:** For each ray, maintain the previous sample's normal $\mathbf{n}_{prev}$ and position to estimate curvature [G].

**Approximation of Normal Curvature ($\kappa_v$):**
Exact computation of $\kappa_v$ via the geometry tensor (derived from the Hessian matrix) is computationally prohibitive for fragment programs and requires significant pre-computation storage [S1]. Instead, $\kappa_v$ is approximated by the rate of change of the normal direction along the viewing ray [S1]:
$$ \kappa_v \approx \frac{\theta}{\Delta s} $$
where $\theta$ is the angle between the normal at the current sample point $\mathbf{n}_{curr}$ and the normal at the previous sample point $\mathbf{n}_{prev}$ along the viewing ray, and $\Delta s$ is the step size between samples [S1]. This approximation is sufficient given the finite resolution of volumetric data and introduces negligible additional cost since normals are already computed per sample [S1].

**Contour Detection Condition:**
A sample is classified as being on a contour if the following condition is met [S1]:
$$ |\mathbf{n} \cdot \mathbf{v}| \le \sqrt{T \kappa_v (2 - T \kappa_v)} $$
where:
*   $\mathbf{n}$ is the surface normal at the sample point [S1].
*   $\mathbf{v}$ is the view vector [S1].
*   $T$ is the user-defined thickness value [S1].
*   $\kappa_v$ is the approximated normal curvature along the view direction [S1].

**Style Integration (Lit Sphere Map Adjustment):**
To allow contour color to vary based on style, the method adjusts the lookup coordinates for the style transfer function based on the curvature measure [S1]. If a sample falls below the contour threshold, the projected normal coordinates are pushed outwards along the radius of the sphere [S1].

Let $r = |\mathbf{n}_{x,y}|$ be the length of the eye-space normal $\mathbf{n}$ projected onto the lit sphere map (the $x,y$ components) [S1]. The adjusted length $r'$ is calculated as [S1]:
$$ r' = \min(1, \delta r) $$
where the scaling factor $\delta$ is defined as [S1]:
$$ \delta = 1 - \min\left(1, \frac{\sqrt{T \kappa_v (2 - T \kappa_v)} - |\mathbf{n} \cdot \mathbf{v}|}{\sqrt{T \kappa_v (2 - T \kappa_v)}}\right) $$
The adjustment allows contour appearance to vary between different styles and based on normal direction while preserving the curvature-based thickness control [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_curvature_controlled_contours(volume, origin, direction, thickness_T, step_size, max_dist, style_map):
    """
    Renders volume with curvature-controlled style contours.
    Args:
        volume: 3D array supporting scalar and normal lookups.
        origin: Ray origin (x, y, z).
        direction: Ray direction (normalized).
        thickness_T: User-defined thickness parameter.
        step_size: Distance between ray casting samples.
        max_dist: Maximum ray length.
        style_map: Function taking (nx, ny) and returning (r, g, b).
    Returns:
        Image buffer (list of lists of [r, g, b, a]).
    """
    height, width = volume.dimensions
    image = [[[0.0, 0.0, 0.0, 0.0] for _ in range(width)] for _ in range(height)]
    
    for y in range(height):
        for x in range(width):
            ray_color = [0.0, 0.0, 0.0, 0.0]
            alpha = 0.0
            prev_normal = None
            t = 0.0
            
            while t < max_dist:
                sample_pos = (origin[0] + t * direction[0], 
                              origin[1] + t * direction[1], 
                              origin[2] + t * direction[2])
                
                if not volume.inside(sample_pos):
                    break
                
                scalar_val = volume.get_scalar(sample_pos)
                current_normal = volume.get_normal(sample_pos)
                
                # Approximate normal curvature kappa_v
                if prev_normal is not None:
                    dot_prod = max(-1.0, min(1.0, sum(p * c for p, c in zip(prev_normal, current_normal))))
                    angle = math.acos(dot_prod)
                    kappa_v = angle / step_size
                else:
                    kappa_v = 0.0
                
                # Check contour condition
                lhs = abs(sum(n * d for n, d in zip(current_normal, direction)))
                rhs_sq = thickness_T * kappa_v * (2 - thickness_T * kappa_v)
                
                is_contour = False
                if rhs_sq >= 0:
                    rhs = math.sqrt(rhs_sq)
                    if lhs <= rhs:
                        is_contour = True
                
                # Determine color and composite
                if is_contour:
                    nx, ny = current_normal[0], current_normal[1]
                    r = math.sqrt(nx*nx + ny*ny)
                    
                    if rhs > 0:
                        ratio = (rhs - lhs) / rhs
                        delta = 1.0 - min(1.0, ratio)
                    else:
                        delta = 1.0
                    
                    r_prime = min(1.0, delta * r)
                    # Reconstruct adjusted normal for style map lookup
                    if r > 0:
                        adj_nx = nx * (r_prime / r)
                        adj_ny = ny * (r_prime / r)
                    else:
                        adj_nx, adj_ny = 0.0, 0.0
                    
                    contour_color = style_map.lookup(adj_nx, adj_ny)
                    # Standard front-to-back compositing [G]
                    alpha_new = 1.0 - (1.0 - alpha) * 1.0
                    ray_color[0] = ray_color[0] * (1.0 - 1.0) + contour_color[0] * 1.0 * (1.0 - alpha)
                    ray_color[1] = ray_color[1] * (1.0 - 1.0) + contour_color[1] * 1.0 * (1.0 - alpha)
                    ray_color[2] = ray_color[2] * (1.0 - 1.0) + contour_color[2] * 1.0 * (1.0 - alpha)
                    alpha = alpha_new
                    break # Contours typically occlude behind them in this style
                else:
                    vol_color = volume.get_color(scalar_val)
                    vol_opacity = volume.get_opacity(scalar_val)
                    # Standard front-to-back compositing [G]
                    alpha_new = 1.0 - (1.0 - alpha) * (1.0 - vol_opacity)
                    ray_color[0] = ray_color[0] * (1.0 - vol_opacity) + vol_color[0] * vol_opacity * (1.0 - alpha)
                    ray_color[1] = ray_color[1] * (1.0 - vol_opacity) + vol_color[1] * vol_opacity * (1.0 - alpha)
                    ray_color[2] = ray_color[2] * (1.0 - vol_opacity) + vol_color[2] * vol_opacity * (1.0 - alpha)
                    alpha = alpha_new
                
                prev_normal = current_normal
                t += step_size
            
            image[y][x] = ray_color
            
    return image
```

**Parameters:**
*   $T$: User-defined thickness value for the contours [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The approximation of $\kappa_v$ using discrete steps is coarse but sufficient for finite-resolution volumes [S1].
*   **Boundary Conditions:** The first sample on a ray has no previous normal; $\kappa_v$ is set to $0.0$ [S1].
*   **Mathematical Validity:** The term under the square root, $T \kappa_v (2 - T \kappa_v)$, must be non-negative [S1]. If $T \kappa_v > 2$, the condition cannot be met, effectively disabling contours for extremely high curvature or large thickness values [S1].

## Subtopics

### Curvature-Based Feature Contouring
A technique that uses transfer functions based on surface curvature to render feature contours with roughly constant image-space thickness [S2]. It is specifically applied to clarify the shape and position of individual bones in small animal imaging, such as mouse embryos [S2]. Unlike the general style transfer approach, it focuses on anatomical feature clarity and is often combined with opacity functions and depth cueing in a multi-dimensional transfer function [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The approximation of $\kappa_v$ adds negligible computational cost compared to standard volume ray casting, as it only requires storing the previous normal and computing an angle [S1]. This allows for interactive performance, unlike exact curvature methods [S1].
*   **Memory:** No additional 3D textures are required for pre-computation (unlike the geometry tensor approach), making it memory-efficient [S1].

**Measured Performance:**
*   The method enables interactive rendering speeds on standard hardware, as it avoids expensive Hessian matrix computations [S1].
*   It successfully renders complex structures like mouse embryo bones with clear feature separation [S2].

**Strengths and Limitations:**
*   **Strengths:** Provides constant visual thickness for contours regardless of surface flatness or fineness; computationally efficient; integrates well with style transfer functions [S1].
*   **Limitations:** The curvature approximation is not exact as it does not step along the iso-surface but along the viewing ray [S1]. It may produce artifacts if the step size is too large relative to the curvature changes [S1].

**When to Use:**
*   Use when consistent contour thickness is critical for perceptual clarity, especially in volumes with mixed flat and fine structures [S1].
*   Use when interactive performance is required and exact curvature computation is too expensive [S1].

**When Not to Use:**
*   Avoid if exact geometric curvature is required for scientific accuracy, as the approximation may deviate from true values [S1].
*   Not suitable if the volume resolution is extremely high and step sizes are very small, potentially leading to noisy curvature estimates [S1].

## Background
Curvature-Controlled Style Contours was proposed by Kindlmann et al. in 2003 [S1]. The method was introduced to address the limitations of standard dot-product contours in volume rendering, which suffer from uncontrolled thickness variations [S1]. It was further refined and implemented in the context of style transfer functions by Bruckner and Gröller in 2007 [S1]. The technique has been applied in scientific visualization, particularly in small animal imaging to clarify bone structures in embryos [S2].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
[S2] Scientific Visualization in Small Animal Imaging, https://sci.utah.edu/scientific-visualization-in-small-animal-imaging/, fetched 2026-10-27
