# Art-Directable Multiple Volumetric Scattering

Also called: Art-Directable Multi-bounce Scattering

## Overview
Art-Directable Multiple Volumetric Scattering is a path tracing method that computes efficient, physically accurate multi-bounce diffusion in participating media while allowing independent artistic control over image contrast [S1]. It distinguishes itself from standard volumetric path tracers by decoupling the physical diffusion (scale) from the visual contrast, enabling biased but convergent rendering of high-order scattering without pre-computation or caching [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera ray, volume density field, extinction coefficients ($\sigma_t, \sigma_s$), phase function $p$, light sources, random number generator, contrast control parameter $a_i$, and bounce depth limit [S1].
*   **Outputs:** Radiance $L_o$ for the camera ray [S1].

**Data Structures**
*   **Volume:** A voxel grid storing density values. Access is via interpolated voxel values [S1].
*   **Random Numbers:** A stream of uniform random variables $\xi_0, \xi_1, \xi_2 \in [0, 1)$ [S1].

**Parameters**
*   $a_i$: Absorption/attenuation factor for contrast control at bounce $i$ [S1].
*   Bounce limit: The maximum number of scattering events [S1]. Example: 16 bounces [S1].
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
        *   $b_i$: A bias or weight factor [S1].
        *   $L_{light}(\boldsymbol{\omega}_i)$: Radiance from the light source in direction $\boldsymbol{\omega}_i$ [S1].
        *   $p(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \mathbf{c}_i)$: Phase function value at point $\mathbf{c}_i$ [S1].
        *   $g$: A geometric or normalization factor [S1].
    e.  **Accumulate:** Add $L_i$ to the total radiance [S1].
    f.  **Update:** Set $\boldsymbol{\omega} \leftarrow \mathbf{v}_0$ for the next iteration [S1].
4.  **Return:** The accumulated radiance from all bounces [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def art_directable_multiple_scattering(ray_origin, ray_direction, volume_grid, sigma_t, sigma_s, phase_func, lights, rng, a_i, max_bounces):
    """
    Renders a camera ray using art-directable multiple volumetric scattering.
    """
    total_radiance = [0.0, 0.0, 0.0]
    current_pos = ray_origin
    current_dir = ray_direction

    # Step 1: Initial scatter selection via density sampling [S1]
    # [G] Standard density sampling along ray (Kulla & Fajardo 2012)
    # Simulate sampling a position proportional to local density
    current_pos = ray_origin + 0.5 * ray_direction  # [G] Placeholder for actual density sampling logic

    for i in range(max_bounces):
        # Step 2: Step length selection using local density approximation [S1]
        local_density = volume_grid.interpolate(current_pos)
        if local_density <= 0.0:
            break
        xi_0 = rng.uniform(0.0, 1.0)
        step_length = -math.log(xi_0) / local_density

        # Step 3: Move and compute optical depth along segment [S1]
        next_pos = current_pos + step_length * current_dir
        optical_depth = sigma_t * step_length  # [G] Constant sigma_t approximation for integral

        # Step 4: Scatter direction inversion [S1]
        xi_1 = rng.uniform(0.0, 1.0)
        xi_2 = rng.uniform(0.0, 1.0)
        new_dir = phase_func.inverse(current_dir, xi_1, xi_2)

        # Step 5: Direct light contribution (Contrast Approximation) [S1]
        # [G] Standard shadow ray trace to lights
        light_rad = [1.0, 1.0, 1.0]  # [G] Placeholder for actual light radiance
        phase_val = phase_func.evaluate(current_dir, new_dir, next_pos)
        b_i = 1.0  # [G] Bias factor, not specified in sources
        g = 1.0    # [G] Geometric factor, not specified in sources

        contrast_factor = math.exp(-a_i * optical_depth)
        direct_contrib = contrast_factor * sigma_s * b_i * light_rad * phase_val * g
        total_radiance = [total_radiance[j] + direct_contrib[j] for j in range(3)]

        # Step 6: Update for next bounce [S1]
        current_pos = next_pos
        current_dir = new_dir

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
