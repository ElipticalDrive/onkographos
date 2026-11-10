# Standard Constant Step Ray Marching

Also called: Constant Step Distance Sampling, Constant step ray marching, March Calculation

## Overview
Standard Constant Step Ray Marching is an iterative root-finding technique for determining the intersection of a ray with an implicit surface or evaluating volumetric fields by stepping along the ray at fixed intervals [S20, S29, unofficial]. It solves the problem of rendering implicit surfaces defined by Signed Distance Functions (SDFs) or scalar fields without requiring analytical intersection solutions or polygonal mesh rasterization [S1, S23, unofficial]. Unlike [Sphere Tracing](1674-sphere-tracing.md), which dynamically adjusts step size based on the local SDF value to guarantee no overshooting, this method uses a predefined constant step size $\Delta t$, simplifying implementation but requiring careful tuning to avoid aliasing or missed details [S3, unofficial][S3, unofficial]. It differs from [Step Size Optimization](2054-step-size-optimization.md) by ignoring local geometry constraints for performance, and from [SDF Ray Marching with Grid Lookup](5017-sdf-ray-marching-with-grid-lookup.md) by not using interpolation for precise intersection detection [S1, unofficial].

## How it works
**Inputs and Outputs**:
*   **Inputs**: Ray origin $\mathbf{o}$, normalized ray direction $\mathbf{d}$, fixed step size $\Delta t$, maximum travel distance $t_{max}$, maximum iteration count $N_{max}$, and convergence threshold $\epsilon$ [S20, S29, unofficial].
*   **Outputs**: Intersection distance $t_{hit}$ (or null if missed), and optionally the hit position $\mathbf{p}$ and normal $\mathbf{n}$ [S8, S21, unofficial].

**Data Structures**:
*   **Ray State**: A scalar $t$ representing accumulated distance, initialized to $t_{min}$ (near clipping plane) [S19][S26, unofficial].
*   **Scene Function**: A function $f(\mathbf{p})$ that returns a scalar value. For SDFs, $f(\mathbf{p})$ is the signed distance to the nearest surface; for boolean volumes, it may return a boolean or density [S14, S29, unofficial].

**Numbered Steps**:
1.  **Initialization**: Set accumulated distance $t \leftarrow t_{min}$ and step counter $k \leftarrow 0$ [S20, S29, unofficial].
2.  **Position Calculation**: Compute the current sample point $\mathbf{p}_k = \mathbf{o} + t \mathbf{d}$ [S20, S29, unofficial].
3.  **Sampling**: Evaluate the scene function at $\mathbf{p}_k$ to get value $v = f(\mathbf{p}_k)$ [S20, S29, unofficial].
4.  **Termination Check**:
    *   If $v \le 0$ (for SDFs) or $v$ indicates a hit (e.g., density threshold), the ray has intersected the surface. Return $t$ as the intersection distance [S20, S29, unofficial].
    *   If $t > t_{max}$ or $k \ge N_{max}$, the ray has escaped or exceeded resources. Return null/miss [S20, S29, unofficial].
5.  **Step Advancement**: Advance the ray by the constant step size: $t \leftarrow t + \Delta t$ [S20, S29, unofficial].
6.  **Increment**: Increment step counter $k \leftarrow k + 1$ [S20, S29, unofficial].
7.  **Loop**: Repeat from Step 2.

**Formulas**:
The ray position at step $k$ is defined as:
$$ \mathbf{r}(k) = \mathbf{o} + k \Delta t \mathbf{d} $$
where $\mathbf{o}$ is the ray origin, $\mathbf{d}$ is the unit ray direction, $\Delta t$ is the predefined constant step size, and $k \in \{0, 1, 2, ...\}$ is the step number [S20, S29, unofficial].

For SDFs, the intersection is detected when:
$$ f(\mathbf{r}(k)) \le 0 $$
[ S20, S29 ]

**Parameters with Typical Values**:
*   $\Delta t$: Fixed step size. Typical values range from $0.001$ to $0.1$ depending on scene scale and desired precision [S7, S14, S24, unofficial].
*   $t_{max}$: Maximum ray distance. Typical values: $10.0$ to $100.0$ [S7, S9, unofficial].
*   $N_{max}$: Maximum iterations. Typical values: $50$ to $255$ [S10, S28, unofficial].
*   $\epsilon$: Convergence threshold. Typical value: $0.001$ [S21, S28, unofficial].

**Edge Cases and Pitfalls**:
*   **Overshooting**: If $\Delta t$ is too large relative to the scene's high-frequency details, the ray may jump over thin objects or surfaces, resulting in missed intersections [S20, S29, unofficial].
*   **Aliasing**: Large $\Delta t$ can cause jagged edges or "staircasing" artifacts on curved surfaces [S3, unofficial].
*   **Performance**: Unlike Sphere Tracing, constant step marching does not adapt to empty space, leading to unnecessary samples in large voids [S3, unofficial].
*   **Precision**: For SDFs, checking $f(\mathbf{p}) \le 0$ may return a point slightly inside the surface. Midpoint approximation ($t - 0.5 \Delta t$) can improve accuracy [S7, unofficial].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def standard_constant_step_ray_march(ray_origin, ray_direction, step_size, max_dist, max_steps, epsilon):
    """
    Performs standard constant step ray marching.
    
    Args:
        ray_origin (vec3): The starting point of the ray.
        ray_direction (vec3): The normalized direction of the ray.
        step_size (float): The fixed distance to advance per iteration (Delta t).
        max_dist (float): The maximum distance the ray can travel.
        max_steps (int): The maximum number of iterations allowed.
        epsilon (float): The threshold for considering a hit (for SDFs, <= 0 is hit).
        
    Returns:
        float or None: The distance to the intersection, or None if no hit is found.
    """
    t = 0.0  # Accumulated distance
    k = 0    # Step counter
    
    # Optional: Start from a near clipping plane if defined
    # t = near_clipping_plane
    
    while k < max_steps:
        # 1. Calculate current position
        current_pos = ray_origin + ray_direction * t
        
        # 2. Evaluate scene function (SDF or density)
        # For SDF: returns distance to nearest surface
        # For boolean/density: returns value indicating inside/outside
        value = scene_function(current_pos)
        
        # 3. Check for intersection
        # For SDFs, intersection occurs when distance <= 0
        # For other fields, check against specific threshold
        if value <= epsilon:
            # Hit detected
            # Optional: Refine position using midpoint approximation
            hit_distance = t - 0.5 * step_size
            return hit_distance
        
        # 4. Check for escape
        if t > max_dist:
            return None
            
        # 5. Advance ray by constant step
        t += step_size
        
        # 6. Increment step counter
        k += 1
        
    # Max steps reached without hit
    return None
```

## Subtopics

### Coarse-to-Fine Entry Point Search
An optimization for sampling within the volume to determine the entry point into solid matter more accurately [S32, unofficial]. It mitigates the inefficiency of constant step marching in large empty spaces by first taking large steps to find the approximate boundary, then switching to smaller constant steps to pinpoint the entry [S32, unofficial].

### Quilez Ray Marching AO
A modification of Evans' method using a ray equation for sampling, allowing for constant step marching along the normal to compute ambient occlusion [S29, unofficial]. It samples along the surface normal $\mathbf{n}$ at the hit point $\mathbf{x}$ using the ray equation $\mathbf{r}_n(k) = \mathbf{x} + k \Delta t \mathbf{n}$ [S29, unofficial]. The occlusion is calculated as $AO = 1 - \frac{1}{2} \sum_{k=1}^{k_{max}} \frac{1}{k} (k \Delta t - f(\mathbf{r}_n(k)))$ [S29, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N$ is the number of steps taken. Since $\Delta t$ is constant, $N \approx t_{max} / \Delta t$ [S20, S29, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary space, as only the current position and accumulated distance need to be stored [S20, S29, unofficial].
*   **Performance**: Faster to implement and compute per step than adaptive methods because it avoids branching based on SDF values for step size calculation [S3, unofficial]. However, it is less efficient in terms of total samples for scenes with large empty regions, as it samples uniformly regardless of distance to geometry [S3, unofficial].
*   **Strengths**: Simple implementation, predictable memory usage, no risk of infinite loops due to oscillating step sizes (if $N_{max}$ is enforced) [S20, S29, unofficial].
*   **Limitations**: Prone to overshooting thin geometry if $\Delta t$ is too large; requires manual tuning of $\Delta t$ for different scene scales; less accurate than Sphere Tracing for SDFs [S3, unofficial][S20, S29, unofficial].
*   **When to Use**: Suitable for simple scenes, volumetric rendering where uniform sampling is acceptable, or when hardware constraints favor fixed-step loops over dynamic branching [S3, unofficial][S24, unofficial].
*   **When Not to Use**: When rendering complex SDF scenes with high-frequency details or large empty spaces, where adaptive stepping (Sphere Tracing) is preferred [S3, unofficial].

## Background
Standard Constant Step Ray Marching originated as a fundamental technique for rendering implicit surfaces before the widespread adoption of Signed Distance Functions (SDFs) for adaptive stepping [S3, unofficial]. It is closely related to early ray tracing methods for implicit curves and surfaces, such as those described by Taubin [S3, unofficial]. The method is often contrasted with "Sphere Tracing" (Hart 1995), which uses the SDF value as the step size to guarantee convergence [S3, S1674]. Constant step marching is still widely used in volume rendering and simple SDF demos due to its simplicity [S1, S24, unofficial].

## Sources
[S1] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S2] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S3] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S4] AustinGoodman/non-linear-sphere-tracing, https://github.com/AustinGoodman/non-linear-sphere-tracing, fetched 2026-10-26 (unofficial)
[S5] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S6] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S7] Adam-Gleave/sdf-terrain, https://github.com/Adam-Gleave/sdf-terrain, fetched 2026-10-26 (unofficial)
[S8] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S9] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S10] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S11] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
[S12] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S13] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S14] Spheretracing Basics | Ronja's tutorials, https://www.ronja-tutorials.com/post/043-spheretracing-basics/, fetched 2026-10-27 (unofficial)
[S15] Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere ..., https://diglib.eg.org/items/7f21e58c-639e-4359-bf94-414738612006, fetched 2026-10-27
[S16] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1819/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S17] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S18] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S19] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S20] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S21] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S22] jk9g13/LaFortuna-SDF-Ray-marching, https://github.com/jk9g13/LaFortuna-SDF-Ray-marching, fetched 2026-10-27 (unofficial)
[S23] Dionysos199/Ray-marching, https://github.com/Dionysos199/Ray-marching, fetched 2026-10-27 (unofficial)
[S24] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S25] Sample - Ray Marching and Signed Distance Functions, https://github.com/ComputeNode/cyfra/pull/13, fetched 2026-10-27 (unofficial)
[S26] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
[S27] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S28] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
[S29] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S30] SDF Raymarching, https://zone.dog/braindump/sdf_marching/, fetched 2026-10-29 (unofficial)
[S31] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
[S32] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
