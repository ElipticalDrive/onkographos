# Curvature-Controlled Style Contours

Also called: Kindlmann Contours, Normal Curvature Contours

## Overview
Curvature-Controlled Style Contours is a non-photorealistic volume rendering technique that generates outline contours with consistent visual thickness by regulating the detection threshold based on the normal curvature along the view direction [S1]. Unlike standard dot-product contours, which produce uncontrolled thickness variations (too thick on flat surfaces, too thin on fine structures), this method ensures that contours remain perceptually uniform across varying surface geometries [S1]. It builds on [Outline and Contour Generation](4137-outline-and-contour-generation.md) by integrating curvature-aware edge detection with style transfer functions for color mapping.

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric dataset, view vector $\mathbf{v}$, user-defined thickness parameter $T$, and a style transfer function (e.g., lit sphere map).
*   **Output:** Rendered volume image with stylized contours.

**Data Structures:**
*   **Volume Data:** 3D texture or grid storing scalar values and pre-computed or on-the-fly normals $\mathbf{n}$.
*   **Ray Casting State:** For each ray, maintain the previous sample's normal $\mathbf{n}_{prev}$ and position to estimate curvature.

**Approximation of Normal Curvature ($\kappa_v$):**
Exact computation of $\kappa_v$ via the geometry tensor (derived from the Hessian matrix) is computationally prohibitive for fragment programs and requires significant pre-computation storage [S1]. Instead, $\kappa_v$ is approximated by the rate of change of the normal direction along the viewing ray [S1]:
$$ \kappa_v \approx \frac{\theta}{\Delta s} $$
where $\theta$ is the angle between the normal at the current sample point $\mathbf{n}_{curr}$ and the normal at the previous sample point $\mathbf{n}_{prev}$ along the viewing ray, and $\Delta s$ is the step size between samples [S1]. This approximation is sufficient given the finite resolution of volumetric data and introduces negligible additional cost since normals are already computed per sample [S1].

**Contour Detection Condition:**
A sample is classified as being on a contour if the following condition is met [S1]:
$$ |\mathbf{n} \cdot \mathbf{v}| \le \sqrt{T \kappa_v (2 - T \kappa_v)} $$
where:
*   $\mathbf{n}$ is the surface normal at the sample point.
*   $\mathbf{v}$ is the view vector.
*   $T$ is the user-defined thickness value.
*   $\kappa_v$ is the approximated normal curvature along the view direction.

**Style Integration (Lit Sphere Map Adjustment):**
To allow contour color to vary based on style (e.g., highlighting), the method adjusts the lookup coordinates for the style transfer function (such as a lit sphere map) based on the curvature measure [S1]. If a sample falls below the contour threshold (i.e., is on a contour), the projected normal coordinates are pushed outwards along the radius of the sphere [S1].

Let $r = |\mathbf{n}_{x,y}|$ be the length of the eye-space normal $\mathbf{n}$ projected onto the lit sphere map (the $x,y$ components). The adjusted length $r'$ is calculated as [S1]:
$$ r' = \min(1, \delta r) $$
where the scaling factor $\delta$ is defined as [S1]:
$$ \delta = 1 - \min\left(1, \frac{\sqrt{T \kappa_v (2 - T \kappa_v)} - |\mathbf{n} \cdot \mathbf{v}|}{\sqrt{T \kappa_v (2 - T \kappa_v)}}\right) $$
This adjustment ensures that contours in highlight regions or varying normal directions can be colored distinctly while maintaining the curvature-based thickness control [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_curvature_controlled_contours(volume, view_vector, thickness_T, step_size, style_map):
    """
    Renders volume with curvature-controlled style contours.
    
    Args:
        volume: 3D array of scalar values and normals.
        view_vector: 3D vector (normalized).
        thickness_T: User-defined thickness parameter.
        step_size: Distance between ray casting samples.
        style_map: Function or texture for color lookup based on normal.
        
    Returns:
        Image buffer.
    """
    image = empty_buffer()
    
    for each pixel:
        ray = generate_ray(pixel)
        color = vec3(0.0)
        alpha = 0.0
        
        prev_normal = None
        
        for sample_pos along ray:
            if not inside_volume(sample_pos):
                break
                
            # Get scalar value and normal at current sample
            scalar_val = volume.get_scalar(sample_pos)
            current_normal = volume.get_normal(sample_pos)
            
            # Approximate normal curvature kappa_v
            # Angle between previous and current normal
            if prev_normal is not None:
                # Clamp dot product to [-1, 1] for safety
                dot_prod = clamp(dot(prev_normal, current_normal), -1.0, 1.0)
                angle = acos(dot_prod)
                kappa_v = angle / step_size
            else:
                # First sample: no previous normal, assume zero curvature or skip
                kappa_v = 0.0
            
            # Check contour condition
            # |n . v| <= sqrt(T * kappa_v * (2 - T * kappa_v))
            lhs = abs(dot(current_normal, view_vector))
            
            # Handle edge case where T*kappa_v > 2, which makes sqrt argument negative
            # In such cases, the condition is never met (no contour) or handled as non-contour
            rhs_sq = thickness_T * kappa_v * (2 - thickness_T * kappa_v)
            
            is_contour = False
            if rhs_sq >= 0:
                rhs = sqrt(rhs_sq)
                if lhs <= rhs:
                    is_contour = True
            
            # Determine color
            if is_contour:
                # Adjust style map lookup coordinates based on curvature
                # Project normal onto x,y plane (assuming z is view direction in lit sphere space)
                # Note: Implementation details of 'style_map' depend on specific lit sphere definition.
                # Here we assume style_map takes (nx, ny) as input.
                
                nx, ny = current_normal.x, current_normal.y
                r = sqrt(nx*nx + ny*ny)
                
                # Calculate delta
                # delta = 1 - min(1, (rhs - lhs) / rhs)
                if rhs > 0:
                    ratio = (rhs - lhs) / rhs
                    delta = 1.0 - min(1.0, ratio)
                else:
                    delta = 1.0 # Fallback
                
                r_prime = min(1.0, delta * r)
                
                # Reconstruct adjusted normal for style map lookup
                # Assuming style map expects normalized or specific coordinate range
                # This part depends on the specific style transfer function implementation
                adjusted_nx = nx # Example: keep x, adjust magnitude via r_prime if needed
                adjusted_ny = ny
                
                # Lookup color from style map
                contour_color = style_map.lookup(adjusted_nx, adjusted_ny)
                
                # Composite contour color
                # Typically contours are drawn on top or blended
                color = contour_color
                alpha = 1.0 # Contours often opaque or high alpha
            else:
                # Standard volume rendering color
                vol_color = volume.get_color(scalar_val)
                # Blend with existing color (standard volume rendering step)
                # This is a simplified representation; actual blending depends on the parent method
                color = blend(color, vol_color, alpha)
                alpha = blend_alpha(alpha, volume.get_opacity(scalar_val))
            
            prev_normal = current_normal
            
        image[pixel] = color

    return image
```

**Parameters:**
*   $T$: User-defined thickness value for the contours [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The approximation of $\kappa_v$ using discrete steps is coarse but sufficient for finite-resolution volumes [S1].
*   **Boundary Conditions:** The first sample on a ray has no previous normal; $\kappa_v$ is set to 0 or the sample is skipped for contour detection [S1].
*   **Mathematical Validity:** The term under the square root, $T \kappa_v (2 - T \kappa_v)$, must be non-negative. If $T \kappa_v > 2$, the condition cannot be met, effectively disabling contours for extremely high curvature or large thickness values [S1].

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
