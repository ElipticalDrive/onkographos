# Collocated Point Light Source Model


## Overview
This model defines the radiance contribution from an ideal Time-of-Flight (ToF) camera that responds exclusively to light emitted by an infrared (IR) point source collocated with the camera lens, ignoring all ambient illumination [S1, unofficial]. It builds on [Continuous-Wave ToF Phasor Model](9486-continuous-wave-tof-phasor-model.md) by enforcing a strict geometric constraint where the incident light direction $\boldsymbol{\omega}_i$ is identical to the viewing direction $\boldsymbol{\omega}_0$ (backscattering along the same ray), and by incorporating the inverse-square falloff of a point source [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $\mathbf{x}$: Position of the light source/camera [S1, unofficial].
    *   $\mathbf{x}_t$: Position along the ray at parameter $t$ [S1, unofficial].
    *   $I$: Emitted radiant intensity of the source (scalar) [S1, unofficial].
    *   $\sigma(\mathbf{x}_t)$: Extinction coefficient at $\mathbf{x}_t$ [S1, unofficial].
    *   $f_{\text{p}}(\mathbf{x}_t, \boldsymbol{\omega}_i, \boldsymbol{\omega}_0, \mathbf{n}(\mathbf{x}_t))$: Scattering phase function depending on the local surface normal $\mathbf{n}(\mathbf{x}_t)$ [S1, unofficial].
    *   $T(\mathbf{x}, \mathbf{x}_t)$: Transmittance from the source to the point $\mathbf{x}_t$ [S1, unofficial].
    *   $t_n, t_r$: Near and far clipping distances along the ray [S1, unofficial].
*   **Output:**
    *   $L(\mathbf{x}, \boldsymbol{\omega}_0)$: Radiance measured by the camera in direction $\boldsymbol{\omega}_0$ [S1, unofficial].

**Data Structures:**
Not specified in the sources.

**Formulas:**
The model assumes monochromatic radiance at a single wavelength (typically IR for ToF) [S1, unofficial].

1.  **Incident Illumination:**
    The incident radiance $L_i(\mathbf{x}_t, \boldsymbol{\omega}_i)$ from a point source at $\mathbf{x}$ is modeled using a Dirac delta function to restrict illumination to the single direction from the source:
    $$
    L_i(\mathbf{x}_t, \boldsymbol{\omega}_i) = \frac{I}{\|\mathbf{x} - \mathbf{x}_t\|^2} \delta \left( \frac{\mathbf{x} - \mathbf{x}_t}{\|\mathbf{x} - \mathbf{x}_t\|} - \boldsymbol{\omega}_i \right) T(\mathbf{x}, \mathbf{x}_t)
    $$
    Here, $\frac{1}{\|\mathbf{x} - \mathbf{x}_t\|^2}$ represents the inverse square light fall-off, and $\delta(\cdot)$ ensures light arrives only from the direction of the source [S1, unofficial].

2.  **Scattering Phase Function Constraint:**
    Because the source is collocated with the camera, the incident direction $\boldsymbol{\omega}_i$ is equal to the outgoing/viewing direction $\boldsymbol{\omega}_0$ [S1, unofficial]. The phase function is evaluated as:
    $$
    I_S(\mathbf{x}_t, \boldsymbol{\omega}_0) = f_{\text{p}}(\mathbf{x}_t, \boldsymbol{\omega}_0, \boldsymbol{\omega}_0, \mathbf{n}(\mathbf{x}_t)) I
    $$
    This represents the emitted light reflected along the same ray [S1, unofficial].

3.  **Resulting Radiance:**
    Substituting the incident illumination into the volume rendering integral yields the final radiance:
    $$
    L(\mathbf{x}, \boldsymbol{\omega}_0) = \int_{t_n}^{t_r} \frac{T(\mathbf{x}, \mathbf{x}_t)^2}{\|\mathbf{x} - \mathbf{x}_t\|^2} \sigma(\mathbf{x}_t) I_S(\mathbf{x}_t, \boldsymbol{\omega}_0) \, dt
    $$
    Note the squared transmittance $T(\mathbf{x}, \mathbf{x}_t)^2$, accounting for attenuation on both the path from the source to the scattering point and from the scattering point back to the camera [S1, unofficial].

**Pitfalls:**
*   **Forward Scattering:** The model ignores forward scattering, which is reasonable if the scene consists mostly of completely opaque surfaces [S1, unofficial].
*   **Ambient Light:** The model explicitly ignores ambient illumination and assumes the camera and light are at the exact same position [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def collocated_point_light_radiance(x_source, ray_origin, ray_direction, t_near, t_far, 
                                   get_sigma, get_normal, get_phase_function, 
                                   source_intensity):
    """
    Computes radiance for a collocated point light source model.
    
    Args:
        x_source: Vector, position of the light source (and camera).
        ray_origin: Vector, origin of the viewing ray (should equal x_source).
        ray_direction: Vector, direction of the viewing ray.
        t_near: Float, near clipping plane distance.
        t_far: Float, far clipping plane distance.
        get_sigma: Function, returns extinction coefficient sigma(x).
        get_normal: Function, returns surface normal n(x).
        get_phase_function: Function, returns f_p(x, omega_i, omega_o, n).
        source_intensity: Float, I.
        
    Returns:
        Float, radiance L(x, omega_0).
    """
    radiance = 0.0
    num_samples = 100  # Implementation detail not in sources
    dt = (t_far - t_near) / num_samples
    
    for i in range(num_samples):
        t = t_near + (i + 0.5) * dt
        x_t = ray_origin + t * ray_direction
        
        # Distance from source to scattering point
        dist_sq = dot(x_source - x_t, x_source - x_t)
        
        # Avoid division by zero if source is on the ray origin
        if dist_sq < 1e-9:
            continue
            
        # Transmittance T(x, x_t) = exp(-integral_{x}^{x_t} sigma(u) du) is computed via standard quadrature [G].
        T = compute_transmittance(x_source, x_t, get_sigma)
        T_squared = T * T
        
        # Extinction
        sigma = get_sigma(x_t)
        
        # Phase function evaluation with omega_i = omega_o = ray_direction
        normal = get_normal(x_t)
        phase_val = get_phase_function(x_t, ray_direction, ray_direction, normal)
        
        # Source intensity term
        I_S = phase_val * source_intensity
        
        # Integrand: (T^2 / dist^2) * sigma * I_S
        integrand = (T_squared / dist_sq) * sigma * I_S
        
        radiance += integrand * dt
        
    return radiance

def compute_transmittance(x_start, x_end, get_sigma):
    """Standard transmittance calculation [G]."""
    num_samples = 10
    dt = dot(x_end - x_start, x_end - x_start) / num_samples
    integral = 0.0
    for i in range(num_samples):
        u = x_start + (i + 0.5) * dt * (x_end - x_start) / dot(x_end - x_start, x_end - x_start)
        integral += get_sigma(u) * dt
    return exp(-integral)
```

## Operating characteristics
*   **Complexity:** The computational cost is dominated by the numerical integration of the volume rendering integral [G]. It requires evaluating transmittance, extinction, and phase functions at each sample point along the ray [G].
*   **Strengths:** Simple analytical form for idealized active illumination scenarios; correctly models inverse-square falloff and double-pass attenuation [S1, unofficial].
*   **Limitations:** Invalid for scenes with significant ambient light or forward scattering in participating media [S1, unofficial]. Assumes the camera and light are at the exact same position [S1, unofficial].

## Background
This model originates from the **TöRF (Time-of-Flight Radiance Fields)** paper, which extends volume rendering to model images captured with Continuous-Wave ToF cameras [S1, unofficial]. It is used to synthesize dynamic scenes by modeling the specific radiance contributions captured by ToF sensors, distinguishing them from standard RGB ambient illumination [S1, unofficial].

## Sources
[S1] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
