# Single Scattering Approximation

Also called: Single Scattering Integral

## Overview
The Single Scattering Approximation simplifies the volume rendering integral by assuming low albedo and/or density, such that the probability of light scattering more than once is small [S1, book]. Only single scattering is considered, and glow is ignored [S1, book].

## How it works
**Inputs:**
*   Volume extinction coefficient field $\tau(\mathbf{x})$ [S1, book].
*   Light source parameters: intensity $L_0$, direction $\boldsymbol{\omega}_0$ (parallel light at infinite distance) [S1, book].
*   Camera parameters: origin $\mathbf{o}$, direction $\boldsymbol{\omega}$ [G].
*   Background radiance $L_{bg}$ [G].

**Outputs:**
*   Radiance $L_o(\mathbf{o}, \boldsymbol{\omega})$ for each pixel [G].

**Data Structures:**
*   **Volume Grid:** A 3D array storing $\tau(\mathbf{x})$ at voxel centers or corners [G].
*   **Shadow Map/Texture:** A 2D texture or buffer storing the precomputed incident radiance $I(\mathbf{x}, \boldsymbol{\omega}_0)$ for points within the volume [S1, book]. If using the bisecting plane optimization, this is a 2D texture representing the plane perpendicular to the bisector of the viewing and light directions [S1, book].

**Steps:**

1.  **Precompute Incident Radiance (Shadowing):**
    Calculate the light reaching each point $\mathbf{x}$ in the volume from the light source, attenuated by the volume's opacity along the light path [S1, book]. For a parallel light source in direction $\boldsymbol{\omega}_0$, the incident radiance $I(\mathbf{x}, \boldsymbol{\omega}_0)$ is:
    $$ I(\mathbf{x}, \boldsymbol{\omega}_0) = L_0 \exp\left(-\int_0^{D} \tau(\mathbf{x} - s\boldsymbol{\omega}_0) ds\right) $$
    where $D$ is the distance from $\mathbf{x}$ to the entry point of the volume along the ray opposite to $\boldsymbol{\omega}_0$ [S1, book].
    *   *Implementation Note:* For gridded data, this is computed efficiently using texture mapping hardware by moving a sampling plane perpendicular to $\boldsymbol{\omega}_0$ in discrete steps along the light flow direction [S1, book].
    *   *Optimization:* If the sampling plane bisects the angle between the viewing ray $\boldsymbol{\omega}$ and illumination ray $\boldsymbol{\omega}_0$, this computation can be performed in the same pass as volume rendering [S1, book].

2.  **Volume Rendering (Single Scattering Integral):**
    Integrate the scattered light along the viewing ray from $\mathbf{o}$ to the far plane [G]. The outgoing radiance is:
    $$ L_o(\mathbf{o}, \boldsymbol{\omega}) = T(\mathbf{o}, \mathbf{p}_{far}) L_{bg} + \int_{\mathbf{o}}^{\mathbf{p}_{far}} T(\mathbf{o}, \mathbf{x}) \sigma_s(\mathbf{x}) I(\mathbf{x}, \boldsymbol{\omega}_0) d\mathbf{x} $$
    where:
    *   $T(\mathbf{o}, \mathbf{x}) = \exp\left(-\int_{\mathbf{o}}^{\mathbf{x}} \tau(\mathbf{u}) d\mathbf{u}\right)$ is the transmittance from the eye to $\mathbf{x}$ [G].
    *   $\sigma_s(\mathbf{x})$ is the scattering coefficient [G]. In many simple approximations, $\sigma_s$ is assumed constant or proportional to $\tau$ [G].
    *   $I(\mathbf{x}, \boldsymbol{\omega}_0)$ is the precomputed incident radiance from Step 1 [S1, book].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scattering_approximation(volume, light_direction, light_intensity, camera_origin, camera_direction, background_radiance, step_size):
    """
    Renders a volume using the Single Scattering Approximation.
    
    Args:
        volume: Object with .contains(x) and .get_extinction(x) methods.
        light_direction: Unit vector omega_0.
        light_intensity: Scalar L_0.
        camera_origin: Vector o.
        camera_direction: Unit vector omega.
        background_radiance: Scalar L_bg.
        step_size: Scalar dt for numerical integration.
        
    Returns:
        List of radiance values for each sample along the ray.
    """
    import math

    # Step 1: Precompute incident radiance I(x, omega_0)
    # For gridded data, this is computed by moving a sampling plane perpendicular to omega_0
    # in discrete steps along the light flow direction [S1, book].
    def get_incident_radiance(x):
        optical_depth = 0.0
        t = 0.0
        # Trace backwards from x along -light_direction to volume entry
        while volume.contains(x - t * light_direction):
            optical_depth += volume.get_extinction(x - t * light_direction) * step_size
            t += step_size
        return light_intensity * math.exp(-optical_depth)

    # Step 2: Integrate scattered light along the viewing ray
    image = []
    num_samples = 100  # Fixed sample count for demonstration [G]
    T = 1.0  # Initial transmittance from eye
    L = 0.0  # Accumulated radiance

    for i in range(num_samples):
        t = i * step_size
        x = camera_origin + t * camera_direction
        if not volume.contains(x):
            break

        tau = volume.get_extinction(x)
        sigma_s = tau * 0.1  # Scattering coefficient for low albedo [G]
        sigma_t = tau

        # Incremental transmittance update: T *= exp(-sigma_t * dt)
        T *= math.exp(-sigma_t * step_size)

        # Contribution: T(o,x) * sigma_s(x) * I(x, omega_0) * dt
        I_in = get_incident_radiance(x)
        L += T * sigma_s * I_in * step_size

    # Add background radiance attenuated by final transmittance
    L += T * background_radiance
    image.append(L)
    return image
```

**Parameters:**
*   $L_0$: Light intensity. Typical values depend on the scene's dynamic range [G].
*   $\boldsymbol{\omega}_0$: Light direction. Assumed to be a single parallel light source at infinite distance [S1, book].
*   $\tau(\mathbf{x})$: Extinction coefficient. Must be provided for the volume [S1, book].

**Edge Cases and Pitfalls:**
*   **High Albedo/Density:** If the volume has high albedo or density, the single scattering assumption breaks down, leading to underestimation of brightness and incorrect glow effects [S1, book].
*   **Precision:** Incremental transmittance multiplication can underflow to zero for deep volumes, causing premature termination of the ray march [G].
*   **Bisecting Plane Optimization:** If using the bisecting plane optimization, the plane must correctly bisect the angle between viewing and illumination rays to ensure accurate shadow sampling [S1, book].

## Subtopics

### [Eye Scattering Recurrence](5316-eye-scattering-recurrence.md)
Computes light exiting particles toward the viewer using precomputed incident light, differing from forward scattering methods.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Efficient for real-time applications due to the simplicity of the integral and the ability to use texture mapping hardware for shadow computation [S1, book].
*   **Strengths:** Simple to implement, fast, suitable for low-density volumes [S1, book].
*   **Limitations:** Inaccurate for high-density or high-albedo volumes; ignores multiple scattering and glow [S1, book].

## Background
The Single Scattering Approximation is a foundational technique in volume rendering, originating from the work of Kajiya and von Herzen [26] and Bahrens and Rattering [5] [S1, book]. It is widely used in scientific visualization and real-time graphics where computational efficiency is prioritized over physical accuracy for dense media [S1, book].

## Sources
[S1] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
