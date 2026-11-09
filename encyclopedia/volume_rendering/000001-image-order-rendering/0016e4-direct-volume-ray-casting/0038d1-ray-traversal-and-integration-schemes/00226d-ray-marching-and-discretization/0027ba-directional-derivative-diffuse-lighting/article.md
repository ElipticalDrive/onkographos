# Directional Derivative Diffuse Lighting

Also called: Density Gradient Diffuse Approximation, Inigo Quilez's directional derivatives

## Overview
Directional Derivative Diffuse Lighting is an approximation for diffuse lighting in volumetric raymarching that estimates light scattering by comparing density at the current point and an offset point in the light direction, avoiding the need for normal calculation [S1, unofficial]. It sets itself apart from standard surface raymarching normals by requiring only two density samples instead of multiple gradient estimates, making it significantly cheaper for the resource-intensive volumetric loop [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes the current 3D position `p`, a normalized light direction `sunDirection`, and a density function `scene(p)` that returns the scalar density at position `p` as inputs [S1, unofficial]. It outputs a diffuse lighting factor and a linear lighting color vector to modulate the accumulated volume color [S1, unofficial].

**Data Structures:**
No complex data structures are required beyond the density function and standard vector types [S1, unofficial]. The method relies on scalar arithmetic and vector addition [S1, unofficial].

**Parameters:**
*   **Offset Scale:** `0.3` (distance along the light direction for the second sample) [S1, unofficial].
*   **Ambient Color:** `vec3(0.60, 0.60, 0.75)` [S1, unofficial].
*   **Ambient Multiplier:** `1.1` [S1, unofficial].
*   **Diffuse Color:** `vec3(1.0, 0.6, 0.3)` [S1, unofficial].
*   **Diffuse Multiplier:** `0.8` [S1, unofficial].
*   **Clamp Range:** `[0.0, 1.0]` [S1, unofficial].

**Steps:**
1.  Sample the density at the current point `p` [S1, unofficial].
2.  Sample the density at an offset point `p'` located further along the light direction [S1, unofficial].
3.  Calculate the diffuse factor by taking the difference between the two density samples, dividing by the offset scale, and clamping the result to `[0.0, 1.0]` [S1, unofficial].
    *   Formula: `diffuse = clamp((scene(p) - scene(p + 0.3 * sunDirection)) / 0.3, 0.0, 1.0)`
    *   Interpretation: If density increases along the light path (`d_current > d_offset`), the result is positive, indicating more scattering [S1, unofficial]. If density decreases, the result is lower or zero, indicating less scattering [S1, unofficial].
4.  Compute the linear lighting color by mixing ambient light and the calculated diffuse light [S1, unofficial].
    *   Formula: `lin = vec3(0.60, 0.60, 0.75) * 1.1 + vec3(1.0, 0.6, 0.3) * diffuse * 0.8`
5.  Determine the base color of the volume slice by mixing white and black based on the current density `d_current` [S1, unofficial].
    *   Formula: `base_color = mix(vec3(1.0, 1.0, 1.0), vec3(0.0, 0.0, 0.0), d_current)`
6.  Combine the base color with the linear lighting and the density alpha, then accumulate into the final result using the over operator [S1, unofficial].
    *   Formula: `color = vec4(base_color * lin, d_current)`
    *   Accumulation: `color.rgb *= color.a; res += color * (1.0 - res.a);` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def directional_derivative_diffuse_lighting_raymarch(origin, direction, step_size, max_steps, sun_direction, scene_func):
    """
    Marches a ray through a volume applying directional derivative diffuse lighting.
    Args:
        origin (vec3): Ray origin.
        direction (vec3): Ray direction (normalized).
        step_size (float): Constant step size for marching.
        max_steps (int): Maximum number of steps to prevent infinite loops.
        sun_direction (vec3): Normalized light direction vector.
        scene_func (callable): Returns scalar density at a 3D position.
    Returns:
        vec4: Accumulated result (rgb, a).
    """
    # Initialize accumulated result to zero (fixes bug where res was undefined/zeroed incorrectly)
    res = vec4(0.0, 0.0, 0.0, 0.0)
    
    # March along the ray
    for i in range(max_steps):
        p = origin + direction * (i * step_size)
        d_current = scene_func(p)
        
        # Only process if density is present
        if d_current > 0.0:
            # Directional derivative approximation
            offset_scale = 0.3
            p_offset = p + offset_scale * sun_direction
            d_offset = scene_func(p_offset)
            
            # Calculate diffuse factor
            diffuse = (d_current - d_offset) / offset_scale
            if diffuse < 0.0:
                diffuse = 0.0
            elif diffuse > 1.0:
                diffuse = 1.0
                
            # Compute linear lighting color
            ambient = vec3(0.60, 0.60, 0.75) * 1.1
            diffuse_light = vec3(1.0, 0.6, 0.3) * diffuse * 0.8
            lin = ambient + diffuse_light
            
            # Determine base color
            base_color = mix(vec3(1.0, 1.0, 1.0), vec3(0.0, 0.0, 0.0), d_current)
            
            # Form slice color and apply lighting
            color = vec4(base_color * lin, d_current)
            color.rgb *= color.a
            
            # Accumulate using over operator
            res.rgb += color.rgb * (1.0 - res.a)
            res.a += color.a * (1.0 - res.a)
            
        # Exit early if fully opaque
        if res.a >= 1.0:
            break
            
    return res
```

**Edge Cases and Pitfalls:**
*   **Normal Calculation Suboptimality:** Standard normal calculation using `getNormal` requires 4-6 sample points to estimate the gradient, which is resource-intensive given the heavy volumetric loop [S1, unofficial]. This method avoids that by using only 2 samples [S1, unofficial].
*   **Loop Inclusion:** Unlike surface raymarching where diffuse is computed once at the surface, this calculation must be included inside the volumetric raymarching loop because density varies throughout the volume [S1, unofficial].
*   **Precision:** The division by `0.3` can amplify noise if the density function is noisy [S1, unofficial]. Clamping to `[0.0, 1.0]` prevents negative lighting contributions but may clip valid gradients if the density change is very steep [S1, unofficial].
*   **Initialization and Loop Bounds:** The accumulation variable `res` must be explicitly initialized to `vec4(0.0, 0.0, 0.0, 0.0)` before the marching loop, and the loop must iterate over the ray path using the specified step size; failure to do so results in a zero output for all rays [G].

## Operating characteristics
*   **Time Complexity:** O(1) per sample point, requiring exactly 2 density function evaluations per lighting calculation [S1, unofficial]. This is significantly faster than gradient-based normal estimation which requires 4-6 evaluations [S1, unofficial].
*   **Memory Complexity:** O(1) additional memory, as it only stores a few scalar and vector variables [S1, unofficial].
*   **Strengths:** Low computational cost, easy to implement, provides a visually plausible approximation of diffuse lighting in volumes without explicit normals [S1, unofficial].
*   **Limitations:** Approximation may not capture complex scattering effects accurately [S1, unofficial].
*   **When to use:** Real-time volumetric rendering where performance is critical and approximate diffuse lighting is sufficient [S1, unofficial].
*   **When not to use:** Offline rendering requiring high physical accuracy, or volumes with very fine details where the fixed offset might miss significant density changes [S1, unofficial].

## Background
The method is attributed to Inigo Quilez and presented in his article on directional derivatives [S1, unofficial]. It was developed as an alternative to standard normal calculation in volumetric raymarching to reduce the computational cost associated with gradient estimation [S1, unofficial].

## Sources
[S1] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
