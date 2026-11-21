# Discrete Ray-Marching with Forward Scattering Approximation

Also called: Ray-marching with light propagation approximation

## Overview
This method approximates the in-scattering integral for volumetric clouds by discretizing the continuous light propagation equation into a Riemann sum using fixed-step ray marching [S1]. It distinguishes itself from general in-scattering approximations by explicitly assuming that 50% of scattered light propagates in the forward direction within a small solid angle (~0.0001 steradians), ignoring the remaining 50% to reduce computational cost [S1]. It builds on [In-Scattering Integral Approximation via Riemann Sum](1585-in-scattering-integral-approximation-via-riemann-sum.md) by applying the forward scattering assumption and Maclaurin expansion for transmittance [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Voxel grid lighting data, passed to the GPU as a 3D texture [S1].
*   Ray marching step size $\Delta s$ [S1].
*   Asymmetry factor $g$ for the phase function [S1].
*   Surface transmittance parameters [S1].

**Outputs:**
*   Final pixel color $I_{total}$ for each frame buffer point, comprising reflected and scattered light components [S1].

### Data Structures
*   **3D Texture:** Stores voxel grid lighting values [S1]. This structure enables hardware-accelerated trilinear interpolation to sample lighting values along the marching ray [S1].
*   **Ray State:** Contains current position $s$, accumulated transmittance, and accumulated incident light [S1].

### Formulas and Definitions
Let $s$ be the position along the ray, starting from the cloud surface [S1]. Let $L(s_i)$ be the illumination at intermediate point $i$ [S1]. Let $T_i$ be the transmittance from the surface to point $i$ [S1].

1.  **Forward Scattering Assumption:** 50% of light scatters in the forward direction within a small solid angle of approximately 0.0001 steradians. The remaining 50% is ignored [S1].
2.  **Incident Light ($I_{inc}$):** Calculated as the summation of light emitted by scattering from each intermediate point $i$, attenuated by transmittance [S1]:
    $$ I_{inc} = \sum_i L(s_i) \cdot T_i \cdot \Delta s_i $$
3.  **Total Light ($I_{total}$):** The sum of reflected and scattered light, both affected by transmittance from the surface to point $s$ [S1]:
    $$ I_{total} = I_{reflected} + I_{scattered} $$
    Where $I_{reflected}$ depends on illumination $L(s)$ and occluded area, and $I_{scattered}$ includes a phase function $p(\theta)$ [S1].
4.  **Segment Integral Approximation:** The continuous integral is converted to a summation of segment integrals [S1]. The phase term is considered constant along the segment and factored out [S1].
5.  **Differential Transmittance ($dT$):** Denoted as the differential transmittance along the segment [S1].
6.  **Transmittance Approximation:** The exponential function $e^{-\tau}$ is approximated using the first two terms of its Maclaurin series expansion [S1]:
    $$ e^{-\tau} \approx 1 - \tau $$
    This simplifies the segment integral calculation [S1].
7.  **Phase Function:** The Henyey–Greenstein function is used to approximate refraction of light in water droplets [S1]:
    $$ p(\theta) = \frac{1-g^2}{(1+g^2-2g\cos\theta)^{3/2}} $$
    Where $\theta$ is the scattering angle and $g$ is the asymmetry factor [S1].

### Parameters
*   **Step Size ($\Delta s$):** The fixed distance between ray march steps [S1].
*   **Asymmetry Factor ($g$):** Parameter for the Henyey–Greenstein phase function [S1].
*   **Solid Angle:** Assumed to be ~0.0001 steradians for the forward scattering component [S1].

### Pseudocode
The following pseudocode implements the ray-marching pass described in [S1]. It defines all required helpers inline and uses standard vector operations.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intersect_aabb(origin, direction, bounds_min, bounds_max):
    """Standard ray-AABB intersection returning t_min and t_max."""
    t_min = max((bounds_min[i] - origin[i]) / direction[i] for i in range(3))
    t_max = min((bounds_max[i] - origin[i]) / direction[i] for i in range(3))
    return t_min, t_max

def ray_march_cloud(ray_origin, ray_direction, bounds_min, bounds_max, step_size, g, lighting_tex, light_source_pos):
    """
    Renders a cloud volume using discrete ray-marching with forward scattering approximation.
    """
    t_min, t_max = intersect_aabb(ray_origin, ray_direction, bounds_min, bounds_max)
    if t_max < 0:
        return (0.0, 0.0, 0.0)

    s_start = ray_origin + ray_direction * t_min
    s_end = ray_origin + ray_direction * t_max

    total_reflected = (0.0, 0.0, 0.0)
    total_scattered = (0.0, 0.0, 0.0)
    current_transmittance = 1.0

    current_pos = s_start
    while current_pos < s_end:
        # Sample lighting using hardware-accelerated trilinear interpolation [S1]
        illumination = lighting_tex.sample(current_pos)

        light_dir = normalize(light_source_pos - current_pos)
        view_dir = -ray_direction
        cos_theta = dot(light_dir, view_dir)

        numerator = 1.0 - g * g
        denominator = 1.0 + g * g - 2.0 * g * cos_theta
        phase = numerator / (denominator ** 1.5)

        forward_scattering_factor = 0.5
        scattered_contribution = illumination * phase * forward_scattering_factor * current_transmittance * step_size
        total_scattered += scattered_contribution

        # Reflected light depends on illumination and occluded area, affected by transmittance [S1]
        occluded_area_factor = 1.0  # Not given in sources [G]
        reflected_contribution = illumination * occluded_area_factor * current_transmittance * step_size
        total_reflected += reflected_contribution

        # Update transmittance: e^(-tau) approx 1 - tau [S1]
        sigma_t = 1.0  # Extinction coefficient not given in sources [G]
        tau = sigma_t * step_size
        current_transmittance *= (1.0 - tau)

        current_pos += ray_direction * step_size

    return total_reflected + total_scattered
```

### Edge Cases and Pitfalls
*   **Transmittance Approximation:** The method uses the linear Maclaurin approximation $e^{-\tau} \approx 1 - \tau$ for transmittance [S1].
*   **Scattering Loss:** The method ignores 50% of scattered light to save computation [S1].
*   **Missing Details:** Specific numerical stability constraints, extinction coefficient values, and occluded area calculations are not given in the sources [G].

## Operating characteristics

### Time and Memory Complexity
Time and memory complexity are not given in the sources.

### Measured Performance
Measured performance metrics are not given in the sources.

### Strengths and Limitations
*   **Strengths:** Computationally efficient due to the forward scattering approximation (ignoring 50% of scattering) and the Maclaurin series approximation for transmittance [S1].
*   **Limitations:** Less physically accurate due to the 50% scattering loss and the linear transmittance approximation [S1].

### When to Use
When to use is not given in the sources.

### When Not to Use
When not to use is not given in the sources.

## Background

### Origin
The method is described in the paper "Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry" [S1].

### History and Development
The approach builds upon previous work on light propagation and scattering approximations, specifically referencing [13] for the forward scattering assumption and [33] for the segment integral conversion [S1].

### Usage
It is used for GPU volumetric cloud rendering [S1].

## Sources

[S1] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
