# Beam × Beam 1D estimate

Also called: 1D kernel density estimation, Beam × Beam 1D, Beam × Beam 2D, Beam × Beam 2D1 estimate, Transient 1D Kernel Density Estimation

## Overview
The Beam × Beam 1D estimate is a radiance estimation technique for participating media that uses photon beams to avoid ray marching, offering better image quality than photon points while being computationally simpler than the 3D variant [S1]. It differs from its siblings by performing spatial blur over a line (1D kernel) rather than a volume or area, which reduces the temporal function to a Dirac delta, making it the most practical implementation among the Beam × Beam family [S1, S2].

## How it works
**Inputs and Outputs**
*   **Input:** A photon beam $b$ with total flux $\Phi_b$, scattering coefficient $\mu_s$, phase function $\rho(\boldsymbol{\theta}_b)$, radius $R_b$, and temporal emission time $t_b$. The beam is defined by a starting position $\mathbf{x}_b$ and direction $\boldsymbol{\omega}_b$.
*   **Input:** An eye ray defined by position $\mathbf{x}_r$, direction $\boldsymbol{\omega}_c$, and time $t$.
*   **Output:** The estimated radiance $L_{b|1D}$ at the intersection point.

**Data Structures**
The method operates on the parameters of the photon beam and the eye ray. No complex spatial data structures (like 3D grids) are required for the estimation step itself, relying instead on the geometric relationship between the beam and the eye ray.

**Steps and Formulas**
1.  **Geometric Projection:** Identify the point on the photon beam $b$ that is closest to the eye ray or the specific point of interest. In the 1D kernel formulation, the spatial blur is performed over a line. The energy of the beam is spread onto the eye ray at a single point $\mathbf{r}(s_r)$ from a single point of the beam $\mathbf{b}(s_b)$ [S2].
2.  **Temporal Alignment:** Because the spatial blur is 1D (along the line of the beam), the temporal function reduces to a Dirac delta function centered at the beam's emission time [S2].
    $$ W_{1D}(t - t_b) = \delta(t - t_b) $$
    This implies that the estimate is non-zero only if the observation time $t$ matches the beam's emission time $t_b$ [S2].
3.  **Radiance Calculation:** Compute the radiance using the 1D kernel density estimate [S2]:
    $$ L_{b|1D}(\mathbf{x}_r, \boldsymbol{\omega}_c, t) = K_{1D}(R_b) \Phi_b \rho(\boldsymbol{\theta}_b) \mu_s \frac{e^{-\mu_s t_b} e^{-\mu_s s_r}}{\sin \theta_b} \delta(t - t_b) $$
    Where:
    *   $K_{1D}(R_b)$ is the 1D kernel function with radius $R_b$.
    *   $\Phi_b$ is the total flux of the photon beam.
    *   $\rho(\boldsymbol{\theta}_b)$ is the phase function evaluated at the scattering angle $\boldsymbol{\theta}_b$.
    *   $\mu_s$ is the scattering coefficient of the medium.
    *   $t_b$ is the time the photon beam was emitted.
    *   $s_r$ is the distance along the eye ray to the point of estimation.
    *   $\theta_b$ is the angle between the beam direction and the eye ray direction (or the relevant geometric angle defining the projection).
    *   $\delta(t - t_b)$ is the Dirac delta function, enforcing temporal coherence [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def beam_beam_1d_estimate(photon_beam, eye_ray, time):
    """
    Estimate radiance using the Beam x Beam 1D method.
    
    Args:
        photon_beam: Object containing phi (flux), mu_s (scattering coeff), 
                     phase_func (function), radius R_b, t_b (emission time),
                     and geometric properties to determine theta_b and s_r.
        eye_ray: Object containing position, direction, and the specific 
                 point of interest s_r along the ray.
        time: The observation time t.
    
    Returns:
        Radiance L_b|1D.
    """
    # 1. Temporal Check: The 1D kernel reduces temporal function to Dirac delta.
    # If the observation time does not match the beam emission time, radiance is zero.
    if time != photon_beam.t_b:
        return 0.0
    
    # 2. Geometric Parameters
    # Calculate the scattering angle theta_b between the beam direction and eye ray direction.
    # Calculate the distance s_r along the eye ray to the interaction point.
    # Note: Exact geometric derivation of theta_b and s_r depends on the specific 
    # intersection logic between the beam line and eye ray point.
    theta_b = calculate_angle(photon_beam.direction, eye_ray.direction)
    s_r = calculate_distance_along_ray(eye_ray, photon_beam.closest_point)
    
    # 3. Kernel Evaluation
    # K_1D(R_b) is the 1D kernel value at the beam radius.
    # This is typically a normalized function (e.g., uniform or Gaussian) over the radius.
    kernel_value = evaluate_1d_kernel(photon_beam.R_b)
    
    # 4. Radiance Calculation
    # Formula: L = K_1D(R_b) * Phi_b * rho(theta_b) * mu_s * 
    #            (exp(-mu_s * t_b) * exp(-mu_s * s_r) / sin(theta_b))
    
    # Avoid division by zero if theta_b is 0 or pi
    sin_theta = sin(theta_b)
    if abs(sin_theta) < 1e-6:
        return 0.0
        
    attenuation = exp(-photon_beam.mu_s * photon_beam.t_b) * \
                  exp(-photon_beam.mu_s * s_r)
                  
    radiance = kernel_value * photon_beam.phi * \
               photon_beam.phase_func(theta_b) * \
               photon_beam.mu_s * \
               (attenuation / sin_theta)
               
    return radiance
```

**Parameters**
*   $R_b$: Photon beam radius. Determined via a pilot density estimate [S1].
*   Kernel $K_{1D}$: Not explicitly defined in the sources, but implied to be a standard 1D density estimation kernel [S2].

**Edge Cases**
*   **Temporal Mismatch:** If $t \neq t_b$, the result is exactly 0 due to the Dirac delta [S2].
*   **Parallel Rays:** If the eye ray is parallel to the beam direction, $\sin \theta_b \to 0$, leading to a singularity. The pseudocode includes a guard for this [G].

## Subtopics

### Beam × Beam 2D1 estimate
A variant that uses a 2D kernel but integrates along one dimension. It is referenced as Equation 29 in [S1] and is considered one of the most promising choices alongside the 1D estimate.

### Beam × Beam 3D estimate
The full 3D convolution variant. It is theoretically superior but "fairly expensive to implement" due to 3D convolution requirements [S1].

### Photon Points
The baseline method using point-based density estimation. Beam × Beam 1D provides better image quality and avoids the over-blurring associated with points [S1].

## Operating characteristics
*   **Complexity:** The 1D variant is "much more practical to implement" and "computationally simpler to evaluate" than the 3D variant because it avoids expensive 3D convolution [S1].
*   **Performance:** In tests, rendering an artifact-free image with 5k photon beams took roughly equal time to render with 25k photon points, but the beam method avoided over-blurring near light sources [S1].
*   **Quality:** Provides a benefit in image quality over photon points and avoids the noise and over-blurring of point-based reconstructions [S1].
*   **Limitations:** The 1D kernel assumes spatial blur is performed over a line, which may not capture all scattering directions as accurately as 2D or 3D kernels in complex geometries, though it retains the benefits of beam-based estimation [S1, S2].

## Background
The Beam × Beam 1D estimate is part of the theory of volumetric radiance estimation using photon points and beams, introduced by Jarosz et al. [S1]. It builds on the 1D kernel density estimation defined by Jarosz et al. [S2]. The method was developed to address the computational cost of ray marching and the quality limitations of photon points, offering a middle ground with the Beam × Beam 3D estimate [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] Progressive Transient Photon Beams, https://arxiv.org/abs/1805.09562v1, fetched 2026-10-27
