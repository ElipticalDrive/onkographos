# Standard Constant Step Ray Marching

Also called: Constant step ray marching, Fixed step ray marching, Uniform step ray marching

## Overview
Standard Constant Step Ray Marching is a rendering technique that samples a scene defined by Signed Distance Fields (SDFs) or implicit functions by advancing a ray along its direction by a fixed, predefined distance $\Delta t$ at each iteration [S1, S13, S19, S27, unofficial]. Unlike Sphere Tracing, which uses the SDF value as the step size to guarantee no overshooting, this method uses a static step size, requiring the step to be smaller than the smallest feature size in the scene to avoid artifacts [S13, S19, unofficial]. It solves the problem of rendering implicit surfaces without rasterization by iteratively checking for intersections (or density accumulation) at uniform intervals [S1, S23, unofficial]. It is distinguished from its siblings by its simplicity and lack of dependency on local geometry estimates for step sizing, making it suitable for non-SDF height fields or when performance constraints favor fixed-loop unrolling over conditional branching [S13, S19, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ (camera position).
*   Ray direction $\mathbf{d}$ (normalized vector from camera to pixel).
*   Fixed step size $\Delta t$ (constant scalar).
*   Maximum travel distance $t_{max}$ (far clipping plane).
*   Maximum step count $N_{max}$ (iteration limit).
*   Surface function $f(\mathbf{p})$: Returns a scalar value. For SDFs, $f(\mathbf{p})$ is the distance to the nearest surface (positive outside, negative inside, zero on surface). For height fields, $f(\mathbf{p})$ may return height or boolean hit status [S6, S13, S19, unofficial].
*   Precision threshold $\epsilon$ (optional, for SDF convergence).

**Outputs:**
*   Intersection distance $t_{hit}$ (if hit).
*   Hit status (boolean).
*   (Optional) Accumulated radiance/color for volume rendering [S23, unofficial].

**Data Structures:**
*   `Ray`: Struct containing `origin` (vec3), `direction` (vec3), `t` (float, accumulated distance).
*   `SceneHit`: Struct containing `position` (vec3), `normal` (vec3, optional), `distanceTravelled` (float), `hit` (bool) [S4, unofficial].

**Parameters:**
*   $\Delta t$: Typical values range from $0.001$ to $0.1$ depending on scene scale [S6, S13, S23, unofficial].
*   $t_{max}$: Typical values $10.0$ to $100.0$ [S6, S13, unofficial].
*   $N_{max}$: Typical values $10$ to $255$ [S6, S13, S23, unofficial].
*   $\epsilon$: Typical value $0.001$ [S20, S26, unofficial].

**Algorithm:**
1.  Initialize accumulated distance $t = 0.0$ (or near clip $t_{min}$) [S6, S12, unofficial][S14].
2.  Loop for $k = 0$ to $N_{max} - 1$:
    a. Calculate current position: $\mathbf{p} = \mathbf{o} + t \cdot \mathbf{d}$ [S7, unofficial][S14][S19, unofficial].
    b. Evaluate surface function: $val = f(\mathbf{p})$ [S3, unofficial][S14][S17, unofficial].
    c. Check termination conditions:
        i. If $val \le 0$ (for SDFs, hit or inside) OR specific height condition met (e.g., $p_y < f(p_x, p_z)$), the ray has intersected [S6, S19, S27, unofficial].
            *   For SDFs: If $|val| < \epsilon$, consider it a hit. Refine position if necessary (e.g., midpoint approximation $t - 0.5 \Delta t$) [S6, S20, unofficial].
            *   Return hit status and $t$ (or $\mathbf{p}$).
        ii. If $t \ge t_{max}$, the ray has missed. Return miss status [S6, S12, unofficial][S14].
    d. Advance ray: $t \leftarrow t + \Delta t$ [S6, S13, S19, unofficial].
3.  If loop completes without hit, return miss status [S19, S27, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def constant_step_ray_march(ray_origin, ray_direction, scene_func, step_size, max_dist, max_steps, epsilon=0.001):
    """
    Marches a ray using a fixed step size.
    
    Args:
        ray_origin: vec3, camera position.
        ray_direction: vec3, normalized ray direction.
        scene_func: callable, f(p) -> float (SDF value or height).
        step_size: float, constant delta t.
        max_dist: float, far clipping plane.
        max_steps: int, maximum iterations.
        epsilon: float, precision threshold for SDF hits.
        
    Returns:
        tuple: (hit: bool, distance: float, position: vec3)
    """
    t = 0.0
    for i in range(max_steps):
        # 1. Calculate current position
        p = ray_origin + ray_direction * t
        
        # 2. Evaluate scene function
        # For SDF: returns distance to surface. < 0 inside, > 0 outside.
        # For Height Field: returns height f(x,z).
        val = scene_func(p)
        
        # 3. Check for intersection
        # Note: Logic varies by function type. 
        # SDF approach: hit if val <= epsilon (surface reached)
        # Height field approach: hit if point is below surface (p.y < val)
        
        is_hit = False
        if val <= epsilon:
            # SDF hit detected
            is_hit = True
        elif scene_func.is_height_field and p.y < val:
            # Height field hit detected
            is_hit = True
            
        if is_hit:
            # Optional refinement for SDF to get closer to zero crossing
            # Midpoint approximation if step was large
            hit_dist = t
            if step_size > epsilon:
                hit_dist = t - 0.5 * step_size
            return (True, hit_dist, p)
            
        # 4. Check for miss (max distance)
        if t >= max_dist:
            return (False, max_dist, p)
            
        # 5. Advance step
        t += step_size
        
    # Max steps reached without hit
    return (False, t, p)
```

**Edge Cases and Pitfalls:**
*   **Overshooting:** If $\Delta t$ is larger than the distance to a thin surface, the ray will pass through it without detection [S19, unofficial]. The step size must be chosen conservatively based on the smallest scene feature [S13, unofficial].
*   **Aliasing:** Large step sizes cause jagged edges and missed details. Smaller steps increase accuracy but reduce performance [S13, S19, unofficial].
*   **Infinite Loops:** If the ray travels parallel to a surface or in empty space, $t$ will eventually exceed $t_{max}$, terminating the loop. A maximum step count $N_{max}$ provides a secondary safety break [S6, S13, unofficial].
*   **Precision:** For SDFs, checking `val <= 0` might miss the exact surface if the step lands slightly outside. Using a small epsilon or refining the hit position (e.g., binary search or midpoint) improves accuracy [S6, S20, unofficial].

## Subtopics

### Coarse-to-Fine Entry Point Search
An optimization for volume rendering where the entry point into solid matter is determined more accurately. It first samples forward along the ray in large steps to find the approximate entry into solid matter, then samples backward in smaller steps to pinpoint the entry point precisely [S30, unofficial].

### Quilez Ray Marching AO
A modification of Evans' method for Ambient Occlusion using constant step marching along the surface normal. It samples points $r_n(k) = x + k \Delta t n$ and calculates occlusion based on the deviation of the sampled distance from the ideal linear progression [S27, unofficial].

### [Iteration-Based Coloring](1244-iteration-based-coloring.md)
A rendering technique where the final pixel color is determined by the number of raymarching steps taken before termination. It visualizes computational intensity, highlighting regions where the ray traveled further or required more sampling iterations.

### [Step Size Selection](2331-step-size-selection.md)
While this article focuses on *constant* step size, the selection of that constant is critical. It balances rendering speed against the accuracy of the Riemann sum approximation. A fixed step size simplifies implementation but requires heuristic selection to avoid aliasing or missed features, unlike adaptive methods.

### [SDF Ray Marching with Grid Lookup](5017-sdf-ray-marching-with-grid-lookup.md)
A GPU-based variant that samples a 3D SDF grid to find intersections. It uses linear interpolation to evaluate the SDF at arbitrary points within the grid bounds, optimizing memory access patterns compared to procedural SDF evaluation.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of steps. Each step involves one function evaluation and one vector addition. The number of steps is proportional to $t_{max} / \Delta t$ [S13, S19, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space per ray. No additional data structures are required beyond the ray state and scene function [S13, unofficial].
*   **Performance:** Generally faster than adaptive sphere tracing for simple scenes or when the step size is small relative to the scene scale, due to predictable memory access and lack of branching on step size calculation [S13, unofficial]. However, it requires more steps than sphere tracing to achieve the same geometric accuracy for complex, varying-scale scenes [S19, unofficial].
*   **Strengths:** Simple implementation, easy to parallelize, no dependency on SDF gradient or local feature size estimation, works with non-SDF implicit functions (e.g., height fields) [S13, S19, unofficial].
*   **Limitations:** Prone to overshooting thin geometry, requires manual tuning of $\Delta t$, inefficient in empty space (cannot skip large distances like sphere tracing) [S13, S19, unofficial].
*   **When to use:** Simple scenes with uniform scale, height field rendering, or when the step size can be guaranteed to be smaller than the smallest feature [S13, S19, unofficial].
*   **When not to use:** Complex scenes with varying feature sizes, where sphere tracing or adaptive stepping would significantly reduce sample count [S19, unofficial].

## Background
Standard constant step ray marching is one of the earliest methods for rendering implicit surfaces. It was described by Hart in 1995 as "Sphere Tracing," although Hart's method specifically used the SDF value as the step size. The constant step variant is a simplification often used in educational contexts and for height fields [S3, S13, unofficial]. It is widely used in demoscene graphics and early shader-based ray marching tutorials due to its simplicity [S1, S13, unofficial]. The method is a foundational technique in the family of distance-field ray marching, preceding more advanced adaptive and optimization techniques [S13, S19, unofficial].

## Sources
[S1] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S3] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S6] Adam-Gleave/sdf-terrain, https://github.com/Adam-Gleave/sdf-terrain, fetched 2026-10-26 (unofficial)
[S13] Spheretracing Basics | Ronja's tutorials, https://www.ronja-tutorials.com/post/043-spheretracing-basics/, fetched 2026-10-27 (unofficial)
[S19] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S20] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S23] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S27] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S30] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
