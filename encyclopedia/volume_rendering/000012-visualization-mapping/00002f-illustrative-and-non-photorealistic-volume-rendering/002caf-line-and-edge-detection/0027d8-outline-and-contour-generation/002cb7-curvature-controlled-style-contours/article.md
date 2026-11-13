# Curvature-Controlled Style Contours

Also called: Kindlmann Contours, Normal Curvature Contours

## Overview
Curvature-Controlled Style Contours is a volume rendering technique that regulates contour thickness based on the normal curvature along the view direction to prevent the uncontrolled variation in thickness inherent in standard dot-product silhouette methods [S1]. While standard methods ($v \cdot n \approx 0$) produce contours that are too thick on flat surfaces and too thin on fine structures, this method uses curvature to maintain consistent visual weight across varying surface geometries [S1]. It builds on [Outline and Contour Generation](4137-outline-and-contour-generation.md) by integrating curvature-aware edge detection with style transfer functions.

## How it works
**Inputs and Outputs:**
- Input: Volumetric dataset, view vector $v$, user-defined thickness parameter $T$.
- Output: Rendered image with stylized contours whose thickness is controlled by local curvature.

**Data Structures:**
- Sample point: Position, normal vector $n$, previous normal vector $n_{prev}$ (from the previous sample along the ray).
- Geometry: No pre-computed Hessian or geometry tensor textures are required [S1].

**Approximation of Normal Curvature ($\kappa_v$):**
Exact curvature requires the geometry tensor derived from the Hessian matrix, which is too expensive for fragment programs and requires two additional 3D textures for pre-computation [S1]. Instead, $\kappa_v$ is approximated using the rate of change of the normal along the viewing ray [S1].
1. During ray casting, store the normal $n_{prev}$ at the previous sample point.
2. Calculate the angle between the current normal $n$ and $n_{prev}$.
3. Divide this angle by the step size $\Delta s$ between sample points:
   $$ \kappa_v \approx \frac{\angle(n, n_{prev})}{\Delta s} $$
   This approximation is coarse because the sampling occurs along the ray, not strictly along the iso-surface, but it is sufficient for finite-resolution volumes and introduces negligible computational cost [S1].

**Contour Detection Condition:**
A sample is considered to be on a contour if the following condition is met [S1]:
$$ |n \cdot v| \le \sqrt{T \kappa_v (2 - T \kappa_v)} $$
Where:
- $n$ is the surface normal at the current sample.
- $v$ is the view vector.
- $T$ is the user-defined thickness value.
- $\kappa_v$ is the approximated normal curvature.

**Style Integration (Lit Sphere Map):**
To integrate contours with style transfer functions (e.g., lit sphere maps), the normal projection coordinates are adjusted based on the curvature measure [S1].
1. Let $r = |n_{x,y}|$ be the length of the eye-space normal projected onto the lit sphere map.
2. Calculate the adjustment factor $\delta$:
   $$ \delta = 1 - \min\left(1, \frac{\sqrt{T \kappa_v (2 - T \kappa_v)} - |n \cdot v|}{\sqrt{T \kappa_v (2 - T \kappa_v)}}\right) $$
3. Adjust the projected radius to $r'$:
   $$ r' = \min(1, \delta r) $$
   This pushes the coordinates outwards along the radius if the sample falls below the contour threshold, allowing the contour color to vary based on the normal direction and curvature [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def curvature_controlled_contour(ray_origin, ray_direction, volume, T):
    # Initialize previous normal
    n_prev = None
    
    # Ray casting loop
    for sample in ray_cast(ray_origin, ray_direction, volume):
        n = sample.normal
        pos = sample.position
        
        # Calculate step size from previous position
        if n_prev is not None:
            dist = distance(pos, sample.previous_position)
            # Approximate curvature: angle between normals divided by step size
            # angle_between returns radians
            kappa_v = angle_between(n, n_prev) / dist
        else:
            kappa_v = 0.0
            
        n_prev = n
        
        # Check contour condition
        threshold = sqrt(T * kappa_v * (2 - T * kappa_v))
        dot_product = abs(dot(n, ray_direction))
        
        is_contour = dot_product <= threshold
        
        if is_contour:
            # Adjust lit sphere map coordinates for style transfer
            r = length(n.xy) # Projected normal length
            numerator = sqrt(T * kappa_v * (2 - T * kappa_v)) - dot_product
            denominator = sqrt(T * kappa_v * (2 - T * kappa_v))
            
            # Avoid division by zero if threshold is 0
            if denominator == 0:
                delta = 1.0
            else:
                delta = 1.0 - min(1.0, numerator / denominator)
                
            r_prime = min(1.0, delta * r)
            
            # Use r_prime to lookup color in style transfer function
            color = lookup_style(r_prime)
        else:
            color = lookup_style(r)
            
        accumulate_color(color)
```

**Parameters:**
- $T$: User-defined thickness value for the contours [S1].

**Edge Cases and Pitfalls:**
- If $\kappa_v$ is zero (flat surface), the threshold becomes zero, and the condition $|n \cdot v| \le 0$ implies only perfectly orthogonal normals are contours, which may result in no contours on flat regions if precision issues arise. However, the formula handles $T \kappa_v (2 - T \kappa_v)$; if $\kappa_v=0$, threshold is 0.
- The approximation of $\kappa_v$ is not geometrically exact as it samples along the ray rather than the surface, but this is accepted as sufficient for the method's purpose [S1].

## Subtopics

### Curvature-Based Feature Contouring
A technique using transfer functions based on surface curvature to render feature contours with roughly constant image-space thickness, specifically applied to clarify the shape and position of individual bones in small animal imaging [S2]. It combines curvature-based opacity with depth cueing to enhance structural clarity [S2].

## Operating characteristics
- **Time Complexity:** Low overhead compared to exact curvature methods. The approximation requires only storing the previous normal and computing an angle, avoiding expensive Hessian calculations or additional texture lookups [S1].
- **Memory Complexity:** Minimal additional memory (one previous normal per ray or tile). No pre-computed 3D textures for geometry tensors are needed [S1].
- **Strengths:** Provides consistent contour thickness across varying surface curvatures, addressing the "too thick on flat, too thin on fine" problem of standard dot-product contours [S1]. Interactive performance is maintained [S1].
- **Limitations:** The curvature approximation is coarse and not geometrically exact [S1]. It relies on the step size of the ray casting; varying step sizes may affect the consistency of $\kappa_v$ [S1].
- **When to use:** When illustrative rendering requires consistent visual emphasis on edges regardless of local surface flatness or curvature, and interactive performance is required [S1].
- **When not to use:** When exact geometric curvature is required for scientific measurement, or when pre-computed geometry tensors are already available and performance is not a constraint [S1].

## Background
The method was proposed by Brückner and Gröller in 2007 as an improvement over the original curvature-controlled contours introduced by Kindlmann et al. in 2003 [S1]. Kindlmann et al. identified the thickness variation problem and proposed using the geometry tensor for exact curvature, but noted its high computational cost and memory requirements for volume rendering [S1]. Brückner and Gröller introduced the normal-angle approximation to make the method viable for direct volume rendering without pre-computation [S1]. The approach has been applied in scientific visualization, such as visualizing bone structures in small animal imaging [S2].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
[S2] Scientific Visualization in Small Animal Imaging, https://sci.utah.edu/scientific-visualization-in-small-animal-imaging/, fetched 2026-10-27
