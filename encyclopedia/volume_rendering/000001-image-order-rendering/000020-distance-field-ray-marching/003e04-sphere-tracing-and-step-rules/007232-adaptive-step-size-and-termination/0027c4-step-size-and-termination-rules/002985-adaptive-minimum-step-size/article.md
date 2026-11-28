# Adaptive Minimum Step Size

Also called: Density-based Step Scaling, Distance-based Step Scaling

## Overview
Adaptive Minimum Step Size is a ray marching optimization technique that dynamically increases the minimum sampling interval ($\Delta t_{min}$) based on the ray's distance from the camera and the accumulated density along the path [S1, unofficial]. It solves the problem of excessive sampling in regions where visual detail is less critical (far from the camera) or where light contribution is negligible (behind dense volumes), thereby reducing computational cost without significantly degrading image quality [S1, unofficial]. Unlike fixed step sizes, it allows for coarser sampling in the background and deep within volumetric structures [S1, unofficial].

## How it works
The method modifies the standard sphere tracing or ray marching loop by maintaining a mutable `minStepSize` variable that grows over the course of the ray traversal.

**Inputs and Outputs:**
*   **Inputs:** Ray origin $\mathbf{o}$, normalized direction $\mathbf{d}$, maximum travel distance $t_{max}$, maximum iterations $N_{max}$, convergence threshold $\epsilon$, and scene functions for distance field $f(\mathbf{p})$ and density $\rho(\mathbf{p})$.
*   **Parameters:**
    *   $k_{density}$: `STEP_INSCREASE_OVER_DENSITY`, controlling growth per unit of sample density [S1, unofficial].
    *   $k_{distance}$: `STEP_INCREASE_OVER_DISTANCE`, controlling growth per unit of step size [S1, unofficial].
    *   Initial `minStepSize`: Typically set to a base resolution value [S1, unofficial].
*   **Data Structures:**
    *   `minStepSize`: A scalar float, initialized to the base step size.
    *   `accumulatedDensity`: A scalar float, initialized to 0.0.

**Algorithm Steps:**
1.  Initialize $t = 0.0$, `minStepSize` = $S_{base}$, `accumulatedDensity` = 0.0.
2.  Loop for $i = 0$ to $N_{max}$:
    a.  **Escape Check:** If $t > t_{max}$, terminate (miss).
    b.  **Position:** $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
    c.  **Distance Field Evaluation:** Evaluate $f(\mathbf{p})$.
    d.  **Step Size Calculation:**
        *   If $f(\mathbf{p}) > 0$ (outside volume): $\Delta t = \max(f(\mathbf{p}), \text{minStepSize})$.
        *   If $f(\mathbf{p}) \le 0$ (inside volume): $\Delta t = \text{minStepSize}$ [S1, unofficial].
    e.  **Advance Ray:** $t \leftarrow t + \Delta t$.
    f.  **Density Evaluation (if inside):** If $f(\mathbf{p}_{prev}) \le 0$ (where $\mathbf{p}_{prev}$ was the position before the step, or the current position if evaluated before step):
        *   Evaluate density $\rho(\mathbf{p})$.
        *   Update `accumulatedDensity` $\leftarrow$ `accumulatedDensity` + $\rho(\mathbf{p}) \cdot \Delta t$ [G] (standard volume integration).
        *   **Adaptive Growth (Density):** $\text{minStepSize} \leftarrow \text{minStepSize} + k_{density} \cdot \rho(\mathbf{p})$ [S1, unofficial].
    g.  **Adaptive Growth (Distance):** $\text{minStepSize} \leftarrow \text{minStepSize} + k_{distance} \cdot \Delta t$ [S1, unofficial].
    h.  **Termination:** Check for surface hit ($|f(\mathbf{p})| < \epsilon$) or early exit based on opacity/transmittance.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_min_step_ray_march(ray_origin, ray_direction, max_dist, max_steps, epsilon):
    """
    Ray marching with adaptive minimum step size based on distance and density.
    """
    t = 0.0
    # S1: Base step size initialization
    min_step_size = 0.1  # Example base value, implementation dependent
    accumulated_density = 0.0
    
    # S1: Constants
    k_density = 0.05      # STEP_INSCREASE_OVER_DENSITY
    k_distance = 0.01     # STEP_INCREASE_OVER_DISTANCE
    
    for i in range(max_steps):
        # Escape check
        if t > max_dist:
            return False, t
        
        # Current position
        p = ray_origin + ray_direction * t
        
        # Evaluate distance field
        # Note: S1 specifies using the sample position, not jittered position
        dist_field = scene_distance_field(p)
        
        # Hit check
        if dist_field < epsilon:
            return True, t
            
        # Step size determination
        if dist_field <= 0:
            # Inside cloud: switch to minimum step size
            step_size = min_step_size
        else:
            # Outside cloud: max of distance field and min step size
            step_size = max(dist_field, min_step_size)
            
        # Advance
        t += step_size
        
        # Update adaptive parameters
        # S1: Increase minStepSize based on distance traveled
        min_step_size += k_distance * step_size
        
        # S1: If inside cloud, increase based on sample density
        if dist_field <= 0:
            sample_density = scene_density(p)
            accumulated_density += sample_density * step_size # G: Standard integration
            min_step_size += k_density * sample_density
            
    return False, t
```

**Edge Cases and Pitfalls:**
*   **Overshooting:** If `minStepSize` grows too large, the ray may skip over thin volumetric features or surface boundaries. The constraint $\Delta t = \max(f(\mathbf{p}), \text{minStepSize})$ when outside helps prevent penetrating the surface, but inside the volume, the fixed `minStepSize` must be small enough to capture the finest details required for the desired resolution [S1, unofficial].
*   **Density Integration:** The source notes imply density is sampled at the current position. Care must be taken to ensure the density value used for `minStepSize` growth corresponds to the segment just traversed or the current voxel, depending on the specific integration scheme [S1, unofficial].

## Subtopics

### Transmittance Field Fitting for Light Penetration
This variant modifies the density field itself to control light penetration depth. It copies the original density field into a new "transmittance" field and remaps the density values from the least dense to the most dense, scaling them between 1.75 and 1.0 [S2, unofficial]. This mapping ensures that as density decreases, the effective ray length (or optical depth) increases, allowing more light to reach the bottom of the cloud structure [S2, unofficial]. It is chosen when simulating realistic light scattering in volumetric clouds where standard density fields result in overly dark undersides [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the number of samples per ray compared to fixed-step marching, particularly for rays passing through large empty spaces or dense cores. The growth rate is linear with respect to distance and accumulated density [S1, unofficial].
*   **Memory Complexity:** Minimal overhead; requires storing `minStepSize` and `accumulatedDensity` per ray.
*   **Strengths:** Significantly improves performance for large-scale volumetric scenes (e.g., clouds) by reducing samples in low-detail areas [S1, unofficial].
*   **Limitations:** May introduce aliasing or miss thin features if the growth constants ($k_{density}, k_{distance}$) are too high. Not suitable for scenes requiring uniform high-resolution sampling throughout the volume [S1, unofficial].

## Background
The method was described by Jack Tollenaar in the context of volumetric cloud rendering, specifically as an optimization on top of sphere tracing for distance-field-based clouds [S1, unofficial]. It addresses the visual artifact of dark cloud undersides by combining step size adaptation with transmittance field fitting techniques discussed in related Houdini/Arnold workflows [S2, unofficial].

## Sources
[S1] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S2] Creating Cloudscapes with Houdini and Arnold, https://sergeneren.com/2019/09/07/creating-cloudscapes/, fetched 2026-10-30 (unofficial)
