# Art-Directable Multiple Volumetric Scattering

Also called: Art-Directable Multi-bounce Scattering

## Overview
Art-Directable Multiple Volumetric Scattering is a path tracing method that computes efficient, physically accurate multi-bounce diffusion in participating media while allowing independent artistic control over image contrast [S1]. It distinguishes itself from standard volumetric path tracers by decoupling the physical diffusion (scale) from the visual contrast, enabling biased but convergent rendering of high-order scattering without pre-computation or caching [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera ray, volume density field, extinction coefficients ($\sigma_t, \sigma_s$), phase function $p$, light sources, random number generator, contrast control parameter $a_i$, and bounce depth limit.
*   **Outputs:** Radiance $L_o$ for the camera ray.

**Data Structures**
*   **Volume:** A voxel grid storing density values. Access is via trilinear interpolation [S1].
*   **Random Numbers:** A stream of uniform random variables $\xi_0, \xi_1, \xi_2 \in [0, 1)$ [S1].

**Parameters**
*   $a_i$: Absorption/attenuation factor for contrast control at bounce $i$. Typical values are not specified in the sources, but it acts as a multiplier in the exponent [S1].
*   Bounce limit: The maximum number of scattering events. Example: 16 bounces [S1].
*   Rays per pixel: Example: 256 [S1].

**Algorithm Steps**
1.  **Initial Scatter Selection:** Choose the first scatter event along the camera ray using the density sampling technique from [Kulla and Fajardo 2012] [S1]. This technique samples positions proportional to the local density, avoiding the need for a full Cumulative Distribution Function (CDF) construction [S1].
2.  **Random Walk Initialization:** Set current position $\mathbf{p}$ and direction $\boldsymbol{\omega}$ based on the initial scatter [S1].
3.  **Loop for Bounces ($i = 0$ to $N$):**
    a.  **Step Length Selection:** For subsequent bounces, approximate Woodcock Tracking by setting the maximum extinction $d_{max}$ equal to the local density $d$ at the current position [S1]. Calculate the step length $l$:
        $$ l = \frac{-\ln(\xi_0)}{d} $$
        where $\xi_0$ is a uniform random number [S1].
        *Note:* This is not mathematically rigorous Woodcock tracking but provides a reasonable measure of maximum extinction in the sample region [S1].
    b.  **Move:** Advance position $\mathbf{p} \leftarrow \mathbf{p} + l \cdot \boldsymbol{\omega}$ [S1].
    c.  **Scatter Direction:** Choose the new outgoing direction $\mathbf{v}_0$ by inverting the phase function $p$:
        $$ \mathbf{v}_0 = p^{-1}(\boldsymbol{\omega}, \xi_1, \xi_2) $$
        where $\xi_1, \xi_2$ are uniform random numbers [S1].
    d.  **Direct Light Contribution (Contrast Approximation):** Calculate the direct light contribution $L_i$ at this bounce using the contrast approximation formula [S1]:
        $$ L_i = e^{-a_i \int_0^t \sigma_t(s) ds} \cdot \sigma_s \cdot b_i \cdot L_{light}(\boldsymbol{\omega}_i) \cdot p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{c}_i) \cdot g $$
        Where:
        *   $a_i$: Contrast control parameter at bounce $i$ [S1].
        *   $\int_0^t \sigma_t(s) ds$: Optical depth along the path segment [S1].
        *   $\sigma_s$: Scattering coefficient [S1].
        *   $b_i$: A bias or weight factor (not explicitly defined in the source text, but present in the formula) [S1].
        *   $L_{light}(\boldsymbol{\omega}_i)$: Radiance from the light source in direction $\boldsymbol{\omega}_i$ [S1].
        *   $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{c}_i)$: Phase function value at point $\mathbf{c}_i$ [S1].
        *   $g$: A geometric or normalization factor (not explicitly defined in the source text, but present in the formula) [S1].
    e.  **Accumulate:** Add $L_i$ to the total radiance.
    f.  **Update:** Set $\boldsymbol{\omega} \leftarrow \mathbf{v}_0$ for the next iteration [S1].
4.  **Return:** Sum of all $L_i$ contributions [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def art_directable_multiple_scattering(camera_ray, volume, sigma_t, sigma_s, phase_function, lights, rng, a_i, max_bounces):
    """
    Renders a camera ray using art-directable multiple volumetric scattering.
    
    Args:
        camera_ray: Ray origin and direction.
        volume: Voxel grid with density values.
        sigma_t: Total extinction coefficient.
        sigma_s: Scattering coefficient.
        phase_function: Function p(omega_i, omega_o, c) returning phase value.
        lights: List of light sources.
        rng: Random number generator.
        a_i: Contrast control parameter for the current bounce.
        max_bounces: Maximum number of scattering events.
        
    Returns:
        Total radiance L_o.
    """
    total_radiance = 0.0
    current_ray = camera_ray
    
    # Step 1: Initial scatter event using density sampling [S1]
    # Implementation detail: Use Kulla and Fajardo 2012 density sampling technique.
    # This involves sampling a position along the ray proportional to local density.
    scatter_pos = sample_density_along_ray(current_ray, volume, rng)
    current_ray.origin = scatter_pos
    
    for i in range(max_bounces):
        # Step 2: Step length selection using local density approximation [S1]
        local_density = volume.interpolate(scatter_pos)
        xi_0 = rng.uniform(0, 1)
        step_length = -math.log(xi_0) / local_density
        
        # Move along the ray
        next_pos = scatter_pos + step_length * current_ray.direction
        
        # Step 3: Scatter direction inversion [S1]
        xi_1 = rng.uniform(0, 1)
        xi_2 = rng.uniform(0, 1)
        new_direction = phase_function.inverse(current_ray.direction, xi_1, xi_2)
        
        # Step 4: Calculate direct light contribution with contrast control [S1]
        # Calculate optical depth along the step segment
        # Note: Exact integration of sigma_t(s) requires sampling along the segment.
        # For simplicity, assume constant sigma_t over the step or use midpoint sampling.
        optical_depth = sigma_t * step_length
        
        # Get light radiance from the dominant light source in the new direction
        # Implementation detail: Shadow ray to lights.
        light_radiance = trace_light_shadows(next_pos, new_direction, lights, volume)
        
        # Evaluate phase function
        phase_val = phase_function.evaluate(current_ray.direction, new_direction, next_pos)
        
        # Apply contrast approximation formula [S1]
        # L_i = exp(-a_i * tau) * sigma_s * b_i * L_light * p * g
        # Note: b_i and g are not explicitly defined in the source. 
        # Assuming b_i=1 and g=1 for basic implementation if not specified.
        # [G] Guard: Ensure local_density is not zero to avoid division by zero.
        if local_density == 0:
            break
            
        contrast_factor = math.exp(-a_i * optical_depth)
        direct_contribution = contrast_factor * sigma_s * light_radiance * phase_val
        
        total_radiance += direct_contribution
        
        # Update for next bounce
        scatter_pos = next_pos
        current_ray.direction = new_direction
        
    return total_radiance
```

## Operating characteristics
*   **Time Complexity:** High. Rendering times for a cloud scape with 16 bounces and 256 rays/pixel were 63m 11s, compared to 5m 16s for single scattering and 6m 4s for the contrast approximation alone [S1].
*   **Memory Complexity:** Low. The method requires no pre-computation or caching, making it memory-efficient and suitable for static and animated volumes [S1].
*   **Strengths:** Independent control over contrast and diffusion; works without pre-computation; independent of volume resolution [S1].
*   **Limitations:** Computationally expensive due to the number of indirect rays; the step length approximation is not mathematically rigorous [S1].
*   **When to use:** When artistic control over contrast is needed in high-albedo media like clouds or whitewater, and performance is secondary to visual quality [S1].

## Background
The method was developed by Pixar Animation Studios for use in the production of *The Good Dinosaur* [S1]. It was presented at ACM SIGGRAPH 2015 [S1]. The approach builds on density sampling techniques from [Kulla and Fajardo 2012] [S1].

## Sources
[S1] Art-directable multiple volumetric scattering, https://doi.org/10.1145/2775280.2792512, fetched 2026-10-31
