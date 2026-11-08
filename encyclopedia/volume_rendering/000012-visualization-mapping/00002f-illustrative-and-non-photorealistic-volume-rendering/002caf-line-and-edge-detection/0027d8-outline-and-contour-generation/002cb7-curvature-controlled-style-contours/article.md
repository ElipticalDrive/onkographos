# Curvature-Controlled Style Contours

Also called: Kindlmann Contours, Normal Curvature Contours

## Overview
Curvature-Controlled Style Contours is a line and edge detection method for illustrative volume rendering that regulates contour thickness based on the normal curvature along the view direction [S1]. It solves the problem of uncontrolled thickness variation inherent in standard dot-product contours, which appear too thick on flat surfaces and too thin on fine structures [S1]. It sets itself apart from siblings like [Object Contour NPR](787-object-contour-npr.md) and [Silhouette Rendering](7959-silhouette-rendering.md) by explicitly incorporating second-order geometric information (curvature) to maintain constant visual thickness, whereas those methods typically rely on first-order normal-view angle relationships or topological features [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric scalar field $f(\mathbf{x})$, view vector $\mathbf{v}$, user-defined thickness parameter $T$.
*   **Output:** A color value for each sample point, where contour regions are darkened or styled based on curvature.

**Data Structures:**
*   **Volume Data:** 3D texture or array storing scalar values and pre-computed gradients (normals).
*   **Ray Casting State:** For each ray, the previous sample point's normal $\mathbf{n}_{prev}$ and the current sample point's normal $\mathbf{n}_{curr}$ are required.

**Approximation of Normal Curvature ($\kappa_v$):**
Exact curvature requires the geometry tensor derived from the Hessian matrix, which is too expensive for fragment programs and requires two additional 3D textures for pre-computation [S1]. Instead, $\kappa_v$ is approximated by the rate of change of the normal direction along the viewing ray [S1].
Let $\Delta \theta$ be the angle between the normal at the current sample point $\mathbf{n}_{curr}$ and the previous normal $\mathbf{n}_{prev}$ along the viewing ray. Let $\Delta s$ be the step size (distance between samples).
$$ \kappa_v \approx \frac{\Delta \theta}{\Delta s} $$
where $\Delta \theta = \arccos(\mathbf{n}_{curr} \cdot \mathbf{n}_{prev})$ [S1]. This approximation is coarse because the ray does not step along the iso-surface, but it is sufficient for finite resolution volumes and introduces almost no additional cost [S1].

**Contour Detection Condition:**
A sample is defined to be on a contour if the following condition is met [S1]:
$$ |\mathbf{n} \cdot \mathbf{v}| \le \sqrt{T \kappa_v (2 - T \kappa_v)} $$
where:
*   $\mathbf{n}$ is the normal at the current sample point.
*   $\mathbf{v}$ is the view vector.
*   $T$ is the user-defined thickness value.
*   $\kappa_v$ is the approximated normal curvature.

**Style Integration (Lit Sphere Map Adjustment):**
To integrate contours with style transfer functions (e.g., lit sphere maps), the coordinates for the lookup are adjusted based on the curvature measure [S1]. If a sample falls below the contour threshold (i.e., satisfies the condition above), the coordinates are pushed outwards along the radius of the sphere [S1].
Let $r = |\mathbf{n}_{x,y}|$ be the length of the eye-space normal $\mathbf{n}$ projected onto the lit sphere map (the $x,y$ components). The adjusted length $r'$ is calculated as [S1]:
$$ r' = \min(1, \delta r) $$
where $\delta$ is defined as:
$$ \delta = 1 - \min\left(1, \frac{\sqrt{T \kappa_v (2 - T \kappa_v)} - |\mathbf{n} \cdot \mathbf{v}|}{\sqrt{T \kappa_v (2 - T \kappa_v)}}\right) $$
This adjustment allows for varying contour appearance based on normal direction and curvature, ensuring consistency with the selected style [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def curvature_controlled_contour(ray_origin, ray_direction, volume, step_size, thickness_T):
    # Initialize previous normal to None
    prev_normal = None
    
    # Iterate along the ray
    for t in range(0, ray_length, step_size):
        current_pos = ray_origin + ray_direction * t
        current_val = volume.sample(current_pos)
        
        # Compute current normal from gradient
        current_normal = normalize(volume.gradient(current_pos))
        
        # Approximate curvature kappa_v
        if prev_normal is not None:
            # Angle between previous and current normal
            dot_product = dot(prev_normal, current_normal)
            # Clamp to avoid numerical errors in acos
            dot_product = clamp(dot_product, -1.0, 1.0)
            angle_delta = acos(dot_product)
            kappa_v = angle_delta / step_size
        else:
            kappa_v = 0.0
            
        # Update previous normal for next iteration
        prev_normal = current_normal
        
        # Check contour condition
        # |n . v| <= sqrt(T * kappa_v * (2 - T * kappa_v))
        abs_n_dot_v = abs(dot(current_normal, ray_direction))
        
        # Handle edge case where T * kappa_v > 2, which makes sqrt argument negative
        curvature_term = thickness_T * kappa_v
        if curvature_term < 2.0 and curvature_term >= 0.0:
            threshold = sqrt(curvature_term * (2.0 - curvature_term))
            is_contour = abs_n_dot_v <= threshold
        else:
            is_contour = False
            
        if is_contour:
            # Adjust lit sphere map coordinates
            # Project normal onto x,y plane (assuming z is view direction or similar mapping)
            # Note: The source specifies nx,y as eye-space normal projected.
            # Assuming standard lit sphere mapping where x,y are horizontal/vertical components
            nx = current_normal.x
            ny = current_normal.y
            r = sqrt(nx*nx + ny*ny)
            
            if r > 0: # Avoid division by zero
                # Calculate delta
                numerator = sqrt(curvature_term * (2.0 - curvature_term)) - abs_n_dot_v
                denominator = sqrt(curvature_term * (2.0 - curvature_term))
                
                # Clamp the fraction to [0, 1] for the inner min
                fraction = numerator / denominator
                fraction = clamp(fraction, 0.0, 1.0)
                
                delta = 1.0 - fraction
                
                # Adjusted radius
                r_prime = min(1.0, delta * r)
                
                # Rescale the projected normal to the new radius
                # This modifies the lookup coordinate
                scale_factor = r_prime / r
                adjusted_nx = nx * scale_factor
                adjusted_ny = ny * scale_factor
                
                # Use adjusted_nx, adjusted_ny for style lookup
                color = style_transfer_function(adjusted_nx, adjusted_ny)
            else:
                color = style_transfer_function(0.0, 0.0)
        else:
            # Standard style lookup without curvature adjustment
            color = style_transfer_function(nx, ny)
            
        accumulate_color(color)
```

## Subtopics

### Curvature-Based Feature Contouring
A technique using transfer functions based on surface curvature to render feature contours with roughly constant image-space thickness [S2]. It is specifically applied to clarify the shape and position of individual bones in small animal imaging, such as mouse embryos [S2]. It sets itself apart by focusing on feature clarity in complex anatomical structures rather than general illustrative style [S2]. Choose this when visualizing dense, overlapping structures like bones where constant thickness aids spatial understanding [S2].

## Operating characteristics
**Time and Memory Complexity:**
The method introduces almost no additional computational cost compared to standard normal-based contours because the normal is evaluated at every sample point anyway [S1]. It avoids the expensive reconstruction of second-order derivatives (Hessian) required for exact curvature, which would otherwise require two additional 3D textures for pre-computation [S1].

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Produces contours of constant visual thickness, avoiding the "too thick on flat surfaces" and "too thin on fine structures" issues of standard dot-product contours [S1]. Efficient for direct volume rendering as it avoids Hessian computation [S1].
*   **Limitations:** The curvature approximation is coarse because it steps along the viewing ray rather than the iso-surface [S1]. It is less accurate than exact geometry tensor methods but sufficient for finite resolution volumes [S1].

**When to use it and when not:**
Use it when constant contour thickness is critical for illustrative clarity and interactive performance is required [S1]. Do not use it if exact geometric curvature is required for scientific measurement, as the approximation is not precise [S1].

## Background
The method was proposed by Bruckner and Gröller in 2007 as an improvement over the original curvature-controlled contours by Kindlmann et al. [S1]. The original work by Kindlmann et al. (2003) introduced the curvature-based contour condition but required expensive Hessian computation, making it unsuitable for interactive direct volume rendering [S1]. Hadwiger et al. (2005) circumvented this by restricting the method to iso-surfaces, but Bruckner and Gröller provided a viable solution for direct volume rendering using the normal angle approximation [S1]. The technique is used in illustrative volume rendering to enhance the visual clarity of volumetric data, particularly in medical imaging contexts such as small animal imaging [S2].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
[S2] Scientific Visualization in Small Animal Imaging, https://sci.utah.edu/scientific-visualization-in-small-animal-imaging/, fetched 2026-10-27
