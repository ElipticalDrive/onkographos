# Step Size and Termination Rules

Also called: Ray Marching Parameters and Compositing

## Overview
Step Size and Termination Rules define the control flow logic for ray-marching loops, determining how far a ray advances per iteration ($\Delta t$) and when the loop halts. This family solves the trade-off between computational efficiency (skipping empty space) and visual fidelity (avoiding missed geometry or aliasing). It sets itself apart from specific implementations like [Sphere Tracing](1674-sphere-tracing.md) or [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by providing the general framework for adaptive stepping, early termination, and safety constraints applicable to both Signed Distance Fields (SDFs) and volumetric density fields.

## Shared principles
All methods in this family operate on a ray defined by origin $\mathbf{o}$ and normalized direction $\mathbf{d}$, parameterized by accumulated distance $t$. The core loop evaluates a scene function (SDF, density field, or volume texture) at the current position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ to determine the next step size $\Delta t$ and check for termination.

**Termination Conditions:**
1.  **Surface Hit:** The ray has intersected the target geometry. For SDFs, this is typically $|f(\mathbf{p})| < \epsilon$ or $f(\mathbf{p}) \le 0$. For volumes, this may be a density threshold or opacity accumulation [S1, unofficial].
2.  **Escape/Miss:** The ray has traveled beyond a maximum distance $t_{max}$ or exceeded a maximum iteration count $N_{max}$.
3.  **Early Termination (Volumes):** Accumulated opacity $\alpha_{acc}$ exceeds a threshold (e.g., $1 - \epsilon$), or accumulated transmission falls below a minimum threshold, allowing the ray to stop early as further contributions are negligible [S1, unofficial].

**Step Size Determination:**
The step size $\Delta t$ is computed based on the local scene properties:
*   **SDF-based:** $\Delta t$ is often derived from the SDF value $f(\mathbf{p})$, potentially scaled by a relaxation factor or Lipschitz constant $\lambda$ [S2, unofficial].
*   **Volume-based:** $\Delta t$ may be fixed, adaptive based on density gradients, or constrained by minimum/maximum bounds to prevent aliasing or excessive computation [S2, unofficial].
*   **Safety:** Steps must be chosen to ensure the ray does not penetrate the surface or skip over significant volumetric features, often requiring conservative clamping.

**Pseudocode Structure:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_loop(ray_origin, ray_direction, scene_func, max_dist, max_steps, epsilon):
    """
    Generic ray marching loop with adaptive step size and termination.
    
    Args:
        ray_origin (vec3): The starting point of the ray.
        ray_direction (vec3): The normalized direction of the ray.
        scene_func (function): A function evaluating the scene (SDF or density) at point p.
        max_dist (float): The maximum distance the ray can travel.
        max_steps (int): The maximum number of iterations allowed.
        epsilon (float): The convergence threshold for hit detection.
        
    Returns:
        tuple: (hit: bool, distance: float)
    """
    t = 0.0
    
    for i in range(max_steps):
        # Check for escape
        if t > max_dist:
            return False, t
        
        # Calculate current position
        p = ray_origin + ray_direction * t
        
        # Evaluate scene function
        val = scene_func(p)
        
        # Termination: Hit
        if is_hit(val, epsilon):
            return True, t
        
        # Termination: Early Exit (Volume specific)
        if is_early_exit(val, t):
            return False, t # Or handle partial contribution
            
        # Step Size Calculation
        step = compute_step_size(val, t, i)
        
        # Safety Clamp
        step = max(step, min_step)
        step = min(step, max_step)
        
        t += step
        
    # Max steps reached
    return False, t
```

## Subtopics

### [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md)
Standard Constant Step Ray Marching uses a predefined constant step size $\Delta t$, simplifying implementation but requiring careful tuning to avoid aliasing or missed details. It differs from [Sphere Tracing](1674-sphere-tracing.md) by ignoring local geometry constraints for performance, and from [SDF Ray Marching with Grid Lookup](5017-sdf-ray-marching-with-grid-lookup.md) by not using interpolation for precise intersection detection.

### [Sphere Tracing](1674-sphere-tracing.md)
Sphere Tracing uses the SDF value at the current position as a safe lower bound for the distance to the nearest surface, allowing the ray to "jump" forward by that exact distance. This guarantees that the ray never overshoots or penetrates the surface, provided the function satisfies the Lipschitz condition. It is distinguished from its siblings by its adaptive step sizing which significantly reduces the number of function evaluations required to converge on a surface.

### [Step Size Optimization](2054-step-size-optimization.md)
Step Size Optimization encompasses techniques that dynamically determine the distance $\Delta t$ to advance a ray, replacing fixed-step methods to balance computational cost and visual fidelity. It differs from [Sphere Tracing](1674-sphere-tracing.md) in that while sphere tracing strictly uses the SDF value as a lower bound for the step (often with a relaxation factor), step size optimization includes broader strategies like error estimation, gap limiting, and integration-based adaptive stepping that may not rely solely on the geometric distance bound.

### [Termination Criteria](17794-termination-criteria.md)
Termination criteria define the conditions under which a ray-marching loop halts, ensuring the algorithm returns a valid intersection point or correctly identifies a miss without infinite iteration. It sets itself apart from sibling methods like [Sphere Tracing](1674-sphere-tracing.md) by focusing on the general control flow logic applicable to both Signed Distance Fields (SDFs) and volumetric density fields, rather than the specific geometric step-size constraints of sphere tracing or the fixed-step simplicity of constant marching.

### Adaptive Minimum Step Size
**Definition:** Dynamic adjustment of the minimum step size based on distance from camera and accumulated density to reduce samples in less important areas [S2, unofficial].

**Mechanism:**
*   Inside the cloud (distanceField $\le 0$), increase `minStepSize` by `STEP_INSCREASE_OVER_DENSITY` multiplied by the current `sampleDensity` [S2, unofficial].
*   After advancing the ray, increase `minStepSize` by `STEP_INCREASE_OVER_DISTANCE` multiplied by the `stepSize` used [S2, unofficial].

**Parameters:**
*   `STEP_INSCREASE_OVER_DENSITY`: Constant controlling how much `minStepSize` increases per unit of `sampleDensity` [S2, unofficial].
*   `STEP_INCREASE_OVER_DISTANCE`: Constant controlling how much `minStepSize` increases per unit of `stepSize` (distance traveled) [S2, unofficial].

**Comparison:** Increases step size further from camera (less detail needed) and behind dense areas (less light contribution), unlike fixed step sizes [S2, unofficial].

### Cone Step Mapping
**Definition:** Cone step mapping and relaxed cone step mapping are space leaping acceleration techniques used in ray marching [S3, unofficial].

### Conservative Sphere Tracing Step Sizing
**Definition:** A fix for volumetric cloud rendering where sphere tracing takes overly aggressive step sizes, skipping cloud boundaries and leaving circular/scalloped density artifacts [S4, unofficial].

**Mechanism:**
*   Clamp the pre-computed 2D organic cell SDF in `cloud_weather_bake.comp` to a conservative maximum of `30000.0 * uWorldScale` [S4, unofficial].
*   Reduce the sphere tracing step size multiplier from `0.8` to `0.4` in `atmosphere_lowres.comp` to make the empty space skipping strictly conservative [S4, unofficial].
*   Clamp the empty space skipping step size to a maximum of `2000.0 * worldScale` to prevent overly aggressive jumps that bypass cloud details or boundaries [S4, unofficial].

**Parameters:**
*   SDF Max Clamp: `30000.0 * uWorldScale` [S4, unofficial].
*   Step Size Multiplier: `0.4` (reduced from `0.8`) [S4, unofficial].
*   Step Size Max Clamp: `2000.0 * worldScale` [S4, unofficial].

**Pitfall:** In empty regions of the weather map, the organic cell SDF returned uncapped values (up to 1e14), causing bilinear filtering to interpolate extremely large values near cloud edges. The empty-space skipping sphere tracing loop used an aggressive step multiplier of 0.8 with no upper bound, resulting in rays jumping past the actual cloud surface [S4, unofficial].

## Background
The concept of step size and termination is fundamental to all ray-marching techniques. Early implementations often used fixed steps due to simplicity, but this led to aliasing and missed geometry in complex scenes [S3, unofficial]. The introduction of Sphere Tracing by John Hart provided a mathematically safe adaptive step size based on SDFs [S3, unofficial]. Subsequent developments, such as those in volumetric cloud rendering, introduced adaptive minimum step sizes to optimize performance in large, sparse volumes [S2, unofficial]. Specific implementations, like the conservative step sizing fixes for volumetric clouds, highlight the ongoing need to balance aggressive space-leaping with visual fidelity [S4, unofficial].

## Sources
[S1] steubengineer/imaginaryVolumes, https://github.com/steubengineer/imaginaryVolumes, fetched 2026-10-26 (unofficial)
[S2] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S3] What is Ray Marching? Is Sphere Tracing the same thing?, https://computergraphics.stackexchange.com/questions/161, fetched 2026-10-29 (unofficial)
[S4] Fix volumetric cloud SDF sphere march step sizing, https://github.com/ricecake/boidsish/pull/659, fetched 2026-10-26 (unofficial)
