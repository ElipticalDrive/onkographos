# Volume Single Scattering (Uniform Ray Marching with Point Light Background)

## Overview
This method computes single scattering radiance in a participating medium using uniform ray marching, where the background radiance is computed via a point light model instead of a generic surface object color [S1, unofficial]. It replaces the Monte Carlo integration of its parent method with deterministic uniform sampling along the ray [S1, unofficial].

## How it works
**Inputs:**
- `ray`: Ray origin $O$ and direction $D$.
- `volume`: Homogeneous participating medium.
- `objects`: List of scene geometry primitives.
- `light`: Point light source definition.
- `t_MAX`: Maximum ray distance to the scene boundary or far plane.
- `N`: Number of uniform marching steps.

**Outputs:**
- `Li`: Final accumulated radiance for the pixel.

**Data Structures:**
- `Sphere`: Defined by radius $r$, center $C$, color, exponent, and radiance [S1, unofficial].
- `Ray`: Contains origin and direction vectors [S1, unofficial].
- `Vector3`: 3D floating-point vector type [S1, unofficial].

**Formulas:**
1. **Homogeneous Transmittance:** Computed via `volume->getTransmittanceHomogeneos(t)` [S1, unofficial].
2. **Background Radiance:** Computed via `getPixelColorPointLight(objects, id, t_MAX, ray, light)` [S1, unofficial].
3. **Light Emission at Step:** Computed via `getPixelColorPointLight(objects, -1, t_march, ray, light)` [S1, unofficial].

**Steps:**
1. Calculate the background radiance using the point light function: `background = getPixelColorPointLight(objects, id, t_MAX, ray, light)` [S1, unofficial].
2. Calculate the transmittance from the ray origin to the hit point at distance `t_MAX`: `transmittance = volume->getTransmittanceHomogeneos(t_MAX)` [S1, unofficial].
3. Initialize the accumulated radiance: `Li = background * transmittance` [S1, unofficial].
4. Iterate through `N` uniform steps:
   a. Compute the step distance: `t_march = (i + 0.5) * (t_MAX / N)` [G].
   b. Calculate transmittance from the eye to the current position: `transmittanceFromEye = volume->getTransmittanceHomogeneos(t_march)` [S1, unofficial].
   c. Calculate the light emission at the current position: `Le = getPixelColorPointLight(objects, -1, t_march, ray, light)` [S1, unofficial].
   d. Calculate transmittance from the current position to the light: `transmittanceFromLight = volume->getTransmittanceHomogeneos(x_march, lightHitPoint)` [S1, unofficial].
   e. Accumulate the scattered contribution: `Li += transmittanceFromEye * Le * transmittanceFromLight * (t_MAX / N)` [G]. (The source notes omit the scattering coefficient $\sigma_s$ and step size $\Delta t$ in the accumulation line, but standard volume rendering requires them for physical correctness [G].)

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def getPixelColor_VolumeSingleScatter_point(ray, volume, objects, light, t_MAX, N):
    # 1. Background radiance
    background = getPixelColorPointLight(objects, id, t_MAX, ray, light)
    # 2. Initial transmittance to boundary
    transmittance = volume.getTransmittanceHomogeneos(t_MAX)
    # 3. Initialize accumulated radiance
    Li = background * transmittance
    
    dt = t_MAX / N
    for i in range(N):
        # 4a. Uniform step distance (midpoint sampling)
        t_march = (i + 0.5) * dt
        # 4b. Transmittance from eye to step
        transmittanceFromEye = volume.getTransmittanceHomogeneos(t_march)
        # 4c. Light emission at step
        Le = getPixelColorPointLight(objects, -1, t_march, ray, light)
        # 4d. Transmittance from step to light
        x_march = ray.origin + ray.direction * t_march
        transmittanceFromLight = volume.getTransmittanceHomogeneos(x_march, light.position)
        # 4e. Accumulate scattering (sigma_s absorbed or assumed unity per source notation)
        Li += transmittanceFromEye * Le * transmittanceFromLight * dt
        
    return Li
```

**Edge Cases and Pitfalls:**
- **Precision:** Exponential transmittance can underflow for large optical depths; clamp the exponent to a minimum value like `-50.0` before evaluation [G].
- **Boundary:** If `t_MAX` is not correctly determined (e.g., ray misses all objects), `background` should default to zero or environment color [G].
- **Light Inside Volume:** If the light position lies inside the volume, `transmittanceFromLight` approaches unity; ensure `Le` handles the geometric singularity at the light position [G].

## Operating characteristics
- **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of marching steps [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory per pixel [G].
- **Performance:** Execution time scales linearly with $N$ and the cost of the point light evaluation [S1, unofficial].
- **Strengths:** Deterministic output with no stochastic noise, and straightforward implementation [G].
- **Limitations:** Aliasing artifacts appear if $N$ is too low, and the point light background approximation fails for complex area lighting [S1, unofficial].
- **When to Use:** Real-time volume rendering with simple lighting, educational demonstrations, or baseline comparisons [G].
- **When Not to Use:** High-fidelity offline rendering requiring multiple scattering, or scenes demanding accurate soft shadows from area lights [G].

## Background
This method originates from the repository `wangyuehi/VolumeSingleScattering` [S1, unofficial]. It implements a variant of uniform ray marching where the boundary condition for the radiative transfer equation is explicitly solved using a point light model [S1, unofficial]. The approach simplifies the rendering equation by neglecting multiple scattering events and relying on deterministic sampling [S1, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
