# Uniform Sphere Tracing

Also called: Ray Marching for Volumes, Rustgpu Sphere Tracing Implementation, Uniform Density Volume Rendering, Volume Rendering with Ray Marching (Uniform Sphere)

## Overview
Uniform Sphere Tracing is a volume rendering technique that combines ray marching along a Signed Distance Field (SDF) with optical depth integration to render participating media. It solves the problem of rendering volumetric effects like fog, smoke, or clouds by stepping through a volume, accumulating absorption and scattering, and blending the result with the background. It is distinguished from its parent [Sphere Tracing](2326-sphere-tracing.md) by its specific application to volume rendering with uniform density assumptions and its reliance on fixed or clamped step sizes rather than purely adaptive SDF-driven steps for the optical integration phase, though it uses SDF for surface detection. It differs from [Zeno Implementation](1544-zeno-implementation.md) and [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md) by not focusing on differentiability or specific engine integration, but rather on the fundamental algorithmic structure of uniform density volume rendering via ray marching.

## How it works
**Inputs and Outputs:**
- **Inputs:** Ray origin $O$, ray direction $D$ (normalized), SDF function $f(P)$, maximum iterations $N_{max}$, minimum step size $d_{min}$, maximum step size $d_{max}$, cutoff threshold $d_{cut}$, absorption coefficient $\sigma_a$, volume scatter color $C_{vol}$, background color $C_{bg}$.
- **Outputs:** Final pixel color $C_{final}$, boolean hit flag, final position, iteration count.

**Data Structures:**
- **Sphere Class (for bounding/intersection):**
  - `center`: `vec3` (e.g., $[0, 0, -4]$)
  - `radius`: `float` (e.g., $1.0$)
  - `sigma_a`: `float` (e.g., $0.1$)
  - `scatter`: `vec3` (e.g., $[0.8, 0.1, 0.5]$)
  - `intersect`: Method returning intersection distances $t_0, t_1$.

**Algorithm Steps:**
1. **Ray Initialization:** For each pixel, compute ray origin $O$ and direction $D$.
2. **Bounding Intersection:** Check if the ray intersects the volume's bounding object (e.g., a sphere).
   - If no intersection, return $C_{bg}$.
   - If intersection exists, calculate entry point $P_1 = O + D \cdot t_0$ and exit point $P_2 = O + D \cdot t_1$.
   - Calculate total distance through volume $L = t_1 - t_0$ (assuming normalized $D$).
3. **Ray Marching Loop:**
   - Initialize `iter_count` = 0, `distance` = 0.0001, `current_pos` = $O$.
   - While `iter_count` < $N_{max}$:
     - Update `current_pos` = $O + \text{distance} \cdot D$.
     - Evaluate SDF: $s = f(\text{current\_pos})$.
     - Determine step increment: $d_{step} = \text{clamp}(s, d_{min}, d_{max})$.
     - Update `distance` += $d_{step}$.
     - Check termination: If $s \le d_{cut}$, set `hit_sdf` = true and break loop.
     - Increment `iter_count`.
4. **Optical Integration (Beer's Law):**
   - Calculate transmittance $T$ through the volume based on the distance traveled or total path length $L$.
   - Formula: $T = e^{-L \cdot \sigma_a}$.
5. **Color Blending:**
   - Calculate final color: $C_{final} = T \cdot C_{bg} + (1 - T) \cdot C_{vol}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_sphere_tracing(ray_origin, ray_direction, sdf_func, sphere_obj, 
                           min_step=0.0001, max_step=0.001, cutoff=0.05, 
                           iter_max=1000):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    sdf_func: callable(pos, iter, prior_d) -> float
    sphere_obj: Sphere object with intersect, sigma_a, scatter, center, radius
    """
    # 1. Bounding Intersection
    t0, t1 = sphere_obj.intersect(ray_origin, ray_direction)
    if not t0: # No intersection
        return sphere_obj.background_color, False, ray_origin, 0, 0.0

    # Calculate entry and exit points
    p1 = ray_origin + ray_direction * t0
    p2 = ray_origin + ray_direction * t1
    
    # Distance through volume
    # Note: If ray_direction is normalized, distance is simply t1 - t0
    distance_through_volume = t1 - t0
    
    # 2. Ray Marching for SDF Hit Detection
    iter_count = 0
    hit_sdf = False
    current_pos = ray_origin
    distance = 0.0001 # Initial small step
    
    while iter_count < iter_max:
        current_pos = ray_origin + distance * ray_direction
        
        # Evaluate SDF
        increment = sdf_func(current_pos, iter_count, float('inf'))
        
        # Clamp step size
        clamped_increment = max(min_step, min(increment, max_step))
        
        # Advance distance
        distance += clamped_increment
        
        # Check for surface hit
        if increment <= cutoff:
            hit_sdf = True
            break
            
        iter_count += 1
        
    # 3. Optical Integration
    # Using Beer's Law for uniform density
    sigma_a = sphere_obj.sigma_a
    transmittance = exp(-distance_through_volume * sigma_a)
    
    # 4. Color Blending
    background_color = sphere_obj.background_color
    volume_color = sphere_obj.scatter
    
    final_color = transmittance * background_color + (1.0 - transmittance) * volume_color
    
    return final_color, hit_sdf, current_pos, iter_count, distance
```

**Parameters:**
- `min_step`: Minimum allowed step increment. Typical: $0.0001$ [S1, unofficial].
- `max_step`: Maximum allowed step increment. Typical: $0.001$ for clamped ray marching, or larger for pure sphere tracing [S1, unofficial].
- `cutoff`: Threshold below which the surface is considered hit. Typical: $0.05$ [S1, unofficial].
- `iter_max`: Maximum number of iterations.
- `sigma_a`: Absorption coefficient. Typical: $0.1$ [S2, unofficial].
- `scatter`: Volume scatter color. Typical: $[0.8, 0.1, 0.5]$ [S2, unofficial].
- `background_color`: Background color. Typical: $[0.572, 0.772, 0.921]$ [S2, unofficial].

**Pitfalls:**
- **SDF Scaling:** Scaling the input position vector to the SDF closure (e.g., `v * 10.0`) without adjusting the step size or SDF output accordingly breaks the sphere tracing property, causing the ray to overshoot collisions [S1, unofficial]. The SDF distance does not scale linearly with the ray step in a way that preserves the tracing logic when the position is scaled but the step is not adjusted [S1, unofficial].
- **Step Clamping:** Using very small `max_step` values effectively turns sphere tracing into uniform ray marching, losing the benefit of large steps in empty space [S1, unofficial].

## Subtopics

### [Spectral Tracking](4664-spectral-tracking.md)
Spectral tracking is a volume rendering technique that efficiently handles per-wavelength optical properties by decomposing the medium's scattering and absorption coefficients into spectral components, using a single majorant density to drive the sampling process for all wavelengths simultaneously. Unlike naive delta tracking which handles each wavelength separately, spectral tracking invokes the tracker only once for all wavelengths, computing a spectrally resolved collision weight at each collision to counteract the discrepancy between the single sampling distribution and the true per-wavelength free-path distribution. This reduces the number of tracker invocations and leverages vectorized operations, making it particularly effective for spectrally heterogeneous media.

### [Auto-Relaxed Sphere Tracing](9468-auto-relaxed-sphere-tracing.md)
Auto-Relaxed Sphere Tracing is a robust sphere tracing method that automatically scales step sizes based on data from previous iterations, using a scalar hyperparameter similar to the learning rate of gradient descent methods. It sets itself apart from its parent [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md) by replacing fixed or heuristic step rules with an adaptive scaling mechanism that reduces the frequency of invalid, over-relaxed step sizes, thereby minimizing the need for robustness-preserving fallback steps. Compared to [Spectral Tracking](4664-spectral-tracking.md), it exhibits a smaller effect on performance from its step-scale hyperparameters and performs up to par with or outperforms the most efficient concurrent methods depending on iteration count limits.

### Discontinuity Reduction via Screen-Space Error Tolerance
A technique to reduce visual discontinuities (jagged edges/artifacts) between adjacent rays by allowing a tolerance error in the intersection condition that scales with screen-space distance from the origin [S3, unofficial]. Unlike standard sphere tracing which requires $f(p)=0$, this method accepts a larger error for rays further away (in screen space), reducing step count and discontinuities without affecting visual quality [S3, unofficial]. The update rule becomes $p_{i+1} = p_i + d \cdot (f(p_i) - \text{err}(\|p_i - o\|_2))$, where $\text{err}(\|p_i - o\|_2)$ is an error function based on the screen-space distance from the ray origin $o$ to the current point $p_i$ [S3, unofficial]. This allows for coarser intersection checks at a distance, mitigating the "wavy" artifacts caused by non-continuous step sizes between neighboring pixels [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot I)$, where $N$ is the number of pixels and $I$ is the average number of iterations per ray. The number of iterations depends on the step size and the complexity of the SDF.
- **Memory Complexity:** $O(1)$ per ray, as it only stores current position, distance, and iteration count.
- **Performance:** Performance is highly dependent on the `max_step` parameter. Small `max_step` values (ray marching) result in more iterations but stable convergence. Large `max_step` values (pure sphere tracing) result in fewer iterations but risk overshooting if the SDF is not properly scaled or if the step rule is violated [S1, unofficial].
- **Strengths:** Simple to implement, robust for convex shapes, efficient for uniform density volumes.
- **Limitations:** Struggles with non-convex shapes without additional logic. Sensitive to SDF scaling errors. Discontinuities between adjacent rays can cause visual artifacts if not handled (e.g., via screen-space error tolerance) [S3, unofficial].
- **When to use:** Rendering simple volumetric effects like fog, smoke, or clouds with uniform density. Suitable for real-time applications where performance is critical and visual artifacts can be mitigated.
- **When not to use:** Rendering complex, non-uniform, or highly heterogeneous media where delta tracking or spectral tracking is more appropriate. Not suitable for non-convex shapes without additional handling.

## Background
Uniform Sphere Tracing for volumes is a combination of two classic computer graphics techniques: sphere tracing (introduced by John C. Hart in 1996 for ray marching SDFs) and volume rendering using Beer's Law (a fundamental principle in optics). The specific implementation details, such as the Rust-based closure-based SDF evaluator and the clamping of step sizes, are documented in community resources and stack exchange discussions [S1, unofficial]. The volume rendering aspect, including the use of a sphere object with uniform absorption and scatter coefficients, is detailed in tutorials on volume rendering [S2, unofficial]. The technique has been adapted and discussed in various contexts, including differentiable rendering and optimized ray marching for fractals [S3, unofficial].

## Sources
[S1] Two different, seemingly equivalent implementations of sphere tracing acting differently, https://computergraphics.stackexchange.com/questions/14565/two-different-seemingly-equivalent-implementations-of-sphere-tracing-acting-dif, fetched 2026-10-26 (unofficial)
[S2] 体渲染简明教程 - BimAnt, https://bimant.com/blog/volume-rendering-brief-intro/, fetched 2026-10-30 (unofficial)
[S3] 要約EnhancedSphereTracing～より良いレイマーチングに向けて～, https://qiita.com/Hirai0827/items/eddcb73a1976c3088b88, fetched 2026-10-26 (unofficial)
