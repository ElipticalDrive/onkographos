# Curvature-Controlled Style Contours

Also called: Kindlmann Contours, Normal Curvature Contours

## Overview
Curvature-Controlled Style Contours is a line and edge detection method that regulates contour thickness based on the normal curvature along the view direction to avoid uncontrolled variation in thickness caused by flat vs. fine structures [S1]. Unlike standard dot-product contours which produce uncontrolled thickness (too thick on flat surfaces and too thin on fine structures) [S1], this method uses a curvature-dependent threshold to maintain consistent visual weight across varying surface geometries [S1]. It builds on [Outline and Contour Generation](4137-outline-and-contour-generation.md) by integrating curvature estimates into the contour detection logic and lit sphere map lookups.

## How it works
**Inputs:** Volumetric scalar field, view vector $v$, user-defined thickness parameter $T$, step size $s$ along the viewing ray.
**Outputs:** Contour color for each sample point.

**Data Structures:**
*   **Normal Buffer:** Stores the surface normal $n$ at the current sample point and the previous sample point along the ray.
*   **Lit Sphere Map:** A 2D texture mapping normal directions to colors.

**Approximation of Normal Curvature ($\kappa_v$):**
Exact computation of $\kappa_v$ via the geometry tensor (derived from the Hessian matrix) is computationally expensive for fragment programs and requires pre-computation of two additional 3D textures [S1]. Instead, $\kappa_v$ is approximated by the rate of change of the normal direction along the viewing ray [S1]. For a sample at the current position with normal $n_{curr}$ and the previous sample position with normal $n_{prev}$ separated by step size $s$:
$$ \kappa_v \approx \frac{\| n_{curr} - n_{prev} \|}{s} $$
This approximation is considered sufficient due to the finite resolution of the volume and introduces negligible additional cost as normals are already evaluated per sample [S1].

**Contour Detection:**
A sample is defined to be on a contour if the following condition holds [S1]:
$$ |n \cdot v| \le \sqrt{T \kappa_v (2 - T \kappa_v)} $$
where:
*   $n$ is the normal at the current sample.
*   $v$ is the view vector.
*   $T$ is the user-defined thickness value.
*   $\kappa_v$ is the approximated normal curvature along the view direction.

**Style Integration (Lit Sphere Map Adjustment):**
To allow contour appearance to vary based on style and normal direction, the coordinates for the lit sphere map lookup are adjusted if the sample falls below the contour threshold [S1]. Let $r = \| n_{x,y} \|$ be the length of the eye-space normal $n$ projected onto the lit sphere map (i.e., the magnitude of the x and y components of the normal in eye space) [S1]. The adjusted radius $r'$ is calculated as [S1]:
$$ r' = \min(1, \delta r) $$
where $\delta$ is defined as:
$$ \delta = 1 - \min\left(1, \frac{\sqrt{T \kappa_v (2 - T \kappa_v)} - |n \cdot v|}{\sqrt{T \kappa_v (2 - T \kappa_v)}}\right) $$
This pushes the coordinates outwards along the radius of the sphere, allowing for varying contour appearance between different styles and based on normal direction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def curvature_controlled_contour(prev_normal, current_normal, view_vector, thickness, step_size, lit_sphere_map):
    # 1. Approximate normal curvature along view direction
    # Note: Normals should be normalized before this step
    normal_diff = current_normal - prev_normal
    kappa_v = length(normal_diff) / step_size
    
    # Guard against division by zero or invalid curvature values
    # If kappa_v is 0 or very small, the term under sqrt might be negative or zero
    # The condition |n.v| <= sqrt(...) handles the threshold logic
    
    # 2. Calculate curvature-controlled threshold
    # Term inside sqrt: T * kappa_v * (2 - T * kappa_v)
    # This term must be non-negative for real solutions.
    # If T * kappa_v > 2, the term becomes negative. 
    # In such cases, the square root is undefined. 
    # Based on the formula structure, if the term is negative, 
    # the condition |n.v| <= imaginary is never met for real dot products,
    # implying no contour. However, typically T is small enough.
    # We clamp the term to 0 to avoid NaNs if it goes slightly negative due to precision.
    term = thickness * kappa_v * (2 - thickness * kappa_v)
    if term < 0:
        term = 0.0
        
    threshold = sqrt(term)
    
    # 3. Check contour condition
    dot_product = abs(dot(current_normal, view_vector))
    is_contour = dot_product <= threshold
    
    if is_contour:
        # 4. Adjust lit sphere map coordinates
        # Project normal onto xy plane (eye space)
        normal_xy = vec2(current_normal.x, current_normal.y)
        r = length(normal_xy)
        
        # Avoid division by zero in delta calculation if threshold is 0
        if threshold > 0:
            ratio = (threshold - dot_product) / threshold
            # Clamp ratio to [0, 1] for the inner min
            clamped_ratio = min(1.0, max(0.0, ratio))
            delta = 1.0 - clamped_ratio
        else:
            delta = 1.0
            
        r_prime = min(1.0, delta * r)
        
        # Normalize the xy direction and scale by r_prime
        if r > 0:
            direction_xy = normal_xy / r
            adjusted_uv = vec2(direction_xy.x * r_prime, direction_xy.y * r_prime)
        else:
            adjusted_uv = vec2(0.0, 0.0)
            
        # Lookup color in lit sphere map using adjusted_uv
        # Note: Lit sphere map usually expects normalized coordinates or specific mapping.
        # The text implies pushing outwards along the radius.
        # Assuming lit_sphere_map takes normalized coordinates or handles the radius.
        # If the map expects normalized (x,y) where x^2+y^2 <= 1:
        # We need to ensure the direction is preserved.
        # The text says "push the coordinates outwards along the radius".
        # If r_prime < r, we are actually pulling inwards if delta < 1?
        # Let's re-read: delta = 1 - min(1, ...). 
        # If dot_product is small (close to 0, i.e., orthogonal), ratio is close to 1.
        # min(1, ratio) is 1. delta is 0. r_prime is 0.
        # This pulls the coordinate to the center?
        # Wait, "push outwards" usually means increasing radius.
        # Let's check the formula again.
        # delta = 1 - min(1, (threshold - dot) / threshold)
        # If dot = 0, ratio = 1, min = 1, delta = 0. r' = 0.
        # If dot = threshold, ratio = 0, min = 0, delta = 1. r' = r.
        # So at the edge of the contour (dot=threshold), we use original r.
        # At the center of the contour (dot=0), we use r'=0.
        # This seems to pull INWARDS.
        # However, the text says "push the coordinates outwards".
        # There might be a misunderstanding of "outwards" or the formula interpretation.
        # Or perhaps "outwards" refers to the direction of the normal projection?
        # Regardless, we implement the formula as given.
        
        color = lit_sphere_map.sample(adjusted_uv)
        return color
    else:
        # Not a contour, return standard color or background
        return background_color
```

## Operating characteristics
*   **Time Complexity:** The approximation of $\kappa_v$ adds minimal cost as it only requires the previous normal, which is available during ray casting [S1]. It avoids the expensive Hessian-based geometry tensor computation [S1].
*   **Memory Complexity:** No additional 3D textures are required for pre-computation, unlike the exact geometry tensor method which would need two additional 3D textures [S1].
*   **Strengths:** Provides consistent contour thickness across varying surface curvatures, addressing the deficiencies of standard dot-product contours [S1]. It is efficient enough for interactive performance [S1].
*   **Limitations:** The curvature approximation is coarse because it steps along the viewing ray rather than the iso-surface, but this is deemed sufficient for finite resolution volumes [S1].
*   **When to use:** When illustrative rendering requires consistent contour emphasis on both flat and fine structures [S1].
*   **When not to use:** When exact geometric curvature is required for scientific accuracy rather than stylistic consistency [S1].

## Background
The method was proposed by Kindlmann et al. in [KWTM03] to regulate contours based on normal curvature [S1]. Hadwiger et al. [HSS*05] later circumvented the computational issues of the exact method by restricting themselves to iso-surfaces [S1]. The specific approximation using the angle between subsequent normals along the viewing ray was introduced by Bruckner and Gröller in 2007 [S1].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
