# Atmospheric Scattering Post-Process

Also called: Post-process Atmospheric Rendering, Screen-space Atmospheric Scattering

## Overview
A post-processing technique that renders volumetric atmospheric scattering (Rayleigh, Mie, and Ozone absorption) over a scene by tracing rays through a spherical atmosphere shell, accounting for occlusion by scene geometry [S1, unofficial]. It differs from in-scene atmospheric rendering by operating on the final screen buffer, using the depth map to determine ray termination distances, and applying tonemapping and saturation adjustments to the accumulated light [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Screen color texture, depth texture, camera position, sun position, planet parameters (radius, atmosphere radius), and scattering coefficients.
- **Outputs**: Final rendered color buffer.

**Data Structures**
The method relies on a uniform structure `AtmosphereSettings` containing the following fields [S1, unofficial]:
- `planetRadius`: Radius of the planet in meters.
- `atmosphereRadius`: Radius of the atmosphere in meters.
- `rayleighHeight`: Height falloff of Rayleigh scattering. Default: $8 \times 10^3$.
- `rayleighScatteringCoefficients`: Rayleigh scattering coefficients (R, G, B). Default: $(5.8 \times 10^{-6}, 13.5 \times 10^{-6}, 33.1 \times 10^{-6})$.
- `mieHeight`: Height falloff of Mie scattering. Default: $1.2 \times 10^3$.
- `mieScatteringCoefficients`: Mie scattering coefficients (R, G, B). Default: $(3.9 \times 10^{-6}, 3.9 \times 10^{-6}, 3.9 \times 10^{-6})$.
- `mieAsymmetry`: Mie scattering asymmetry factor, between -1 and 1. Default: $0.8$.
- `ozoneHeight`: Height of the ozone layer in meters above the planet surface. Default: $25 \times 10^3$.
- `ozoneAbsorptionCoefficients`: Ozone absorption coefficients (R, G, B). Default: $(0.6 \times 10^{-6}, 1.8 \times 10^{-6}, 0.085 \times 10^{-6})$.
- `ozoneFalloff`: Ozone absorption falloff around the ozone layer height in meters. Default: $5 \times 10^3$.
- `lightIntensity`: Intensity of the sun. Default: $20$.

**Numbered Steps**

1.  **Sample Input Data**: Read the screen color and depth value for the current pixel UV coordinates [S1, unofficial].
    ```python
    screen_color = texture2D(texture_sampler, uv).rgb
    depth = texture2D(depth_sampler, uv).r
    ```

2.  **Compute Ray Origin and Direction**: Transform the depth value to world space to find the deepest visible point, then compute the ray direction from the camera position [S1, unofficial].
    ```python
    deepest_point = world_from_uv(uv, depth) - camera_position
    maximum_distance = length(deepest_point)
    ray_dir = deepest_point / maximum_distance
    ```

3.  **Clamp Ray Length for Sphere Intersection**: Intersect the ray with the planet sphere to get entry distance $t_0$. If the ray intersects and the computed maximum distance is greater than $t_0 - 1.0$ (to avoid precision artifacts from imperfect sphere discretization), clamp the maximum distance to $t_0$. Note: This clamping logic should not be used if the planet has landmasses or non-spherical geometry [S1, unofficial].
    ```python
    t0, t1 = ray_intersect_sphere(camera_position, ray_dir, planet_position, planet_radius)
    if maximum_distance > t0 - 1.0:
        maximum_distance = t0
    ```

4.  **Atmosphere Intersection**: Call the `scatter` function with the screen color, camera position, ray direction, and maximum distance [S1, unofficial]. Inside `scatter`:
    a. Intersect the ray with the atmosphere sphere to get impact point $t_{in}$ and escape point $t_{out}$ [S1, unofficial].
    b. If no intersection occurs, return the original screen color [S1, unofficial].
    c. Clamp $t_{in}$ to $\max(0.0, t_{in})$ [S1, unofficial].
    d. Clamp $t_{out}$ to $\min(\text{maximum\_distance}, t_{out})$ [S1, unofficial].
    e. Calculate the distance through the atmosphere: $d_{atm} = \max(0.0, t_{out} - t_{in})$ [S1, unofficial].
    f. Compute the first point in the atmosphere: $P_{start} = \text{ray\_origin} + \text{ray\_dir} \times t_{in}$ [S1, unofficial].
    g. Call `calculate_light` with $P_{start}$, ray direction, $d_{atm}$, and original color [S1, unofficial].

5.  **Calculate Scattering (`calculate_light`)**:
    a. Define the number of sample points `POINTS_FROM_CAMERA` (implementation detail, not explicitly defined in sources, but used in loop) [S1, unofficial].
    b. Compute step size: $s = d_{atm} / (\text{POINTS\_FROM\_CAMERA} - 1)$ [S1, unofficial].
    c. Initialize accumulators: `in_scattered_rayleigh` = 0, `in_scattered_mie` = 0, `total_optical_depth` = 0 [S1, unofficial].
    d. Iterate $i$ from 0 to `POINTS_FROM_CAMERA` - 1:
        i. Current sample point: $P_{sample} = P_{start} + \text{ray\_dir} \times (i \times s)$ [S1, unofficial].
        ii. Compute Sun Ray Optical Depth:
            - Calculate distance from light source through atmosphere: $L_{sun} = \text{atmosphere\_radius} - \text{length}(P_{sample} - \text{planet\_position})$ [S1, unofficial].
            - If ray from $P_{sample}$ in sun direction intersects atmosphere sphere, update $L_{sun}$ to the intersection distance $t_1$ [S1, unofficial].
            - Compute optical depth from sun to sample point: $OD_{sun} = \text{optical\_depth}(P_{sample}, \text{sun\_dir}, L_{sun})$ [S1, unofficial].
        iii. Compute View Ray Optical Depth:
            - Distance from sample point to camera along view ray: $L_{view} = i \times s$ [S1, unofficial].
            - Compute optical depth from sample point to camera: $OD_{view} = \text{optical\_depth}(P_{sample}, -\text{ray\_dir}, L_{view})$ [S1, unofficial].
        iv. Compute Transmittance:
            - $T = \exp(-\text{rayleigh\_coeffs} \times (OD_{sun}.x + OD_{view}.x) - \text{mie\_coeffs} \times (OD_{sun}.y + OD_{view}.y) - \text{ozone\_coeffs} \times (OD_{sun}.z + OD_{view}.z))$ [S1, unofficial].
        v. Update Total Optical Depth:
            - Get local density at sample point: $\rho = \text{density\_at\_point}(P_{sample})$ [S1, unofficial].
            - $OD_{total} += \rho \times s$ [S1, unofficial].
        vi. Accumulate Scattered Light:
            - `in_scattered_rayleigh` += $\rho.x \times T \times s$ [S1, unofficial].
            - `in_scattered_mie` += $\rho.y \times T \times s$ [S1, unofficial].

    e. Compute Phase Functions:
        - Cosine of angle between view and sun directions: $c = \text{dot}(\text{ray\_dir}, \text{sun\_dir})$ [S1, unofficial].
        - Rayleigh Phase Function: $P_R = \frac{3}{16\pi} (1 + c^2)$ [S1, unofficial].
        - Mie Phase Function: Let $g = \text{mie\_asymmetry}$.
          $$P_M = \frac{3(1-g^2)}{2(2+g^2)} \frac{1+c^2}{(1+g^2-2gc)^{1.5}}$$
          [S1, unofficial].

    f. Apply Phase Functions and Coefficients:
        - `in_scattered_rayleigh` *= $P_R \times \text{rayleigh\_coeffs}$ [S1, unofficial].
        - `in_scattered_mie` *= $P_M \times \text{mie\_coeffs}$ [S1, unofficial].

    g. Compute Opacity (Transmittance to Camera):
        - $O = \exp(-(\text{mie\_coeffs} \times OD_{total}.y + \text{rayleigh\_coeffs} \times OD_{total}.x + \text{ozone\_coeffs} \times OD_{total}.z))$ [S1, unofficial].

    h. Return Final Color:
        - $\text{final} = (\text{in\_scattered\_rayleigh} + \text{in\_scattered\_mie}) \times \text{light\_intensity} + \text{original\_color} \times O$ [S1, unofficial].

6.  **Post-Processing**:
    a. Apply exposure: $\text{final\_color} \times 1.2$ [S1, unofficial].
    b. Apply ACES tonemapping [S1, unofficial].
    c. Apply saturation (factor 1.2) using grayscale luminance weights $(0.299, 0.587, 0.114)$:
       $$\text{grayscale} = \text{dot}(\text{final\_color}, [0.299, 0.587, 0.114])$$
       $$\text{final\_color} = \text{mix}(\text{grayscale}, \text{final\_color}, 1.2)$$
       [S1, unofficial].
    d. Clamp to $[0.0, 1.0]$ and output [S1, unofficial].

**Formulas**
- **Rayleigh Phase Function**:
  $$P_R(\theta) = \frac{3}{16\pi} (1 + \cos^2 \theta)$$
  where $\theta$ is the scattering angle [S1, unofficial].

- **Mie Phase Function (Henyey-Greenstein variant)**:
  $$P_M(\theta) = \frac{3(1-g^2)}{2(2+g^2)} \frac{1+\cos^2 \theta}{(1+g^2-2g\cos \theta)^{3/2}}$$
  where $g$ is the asymmetry factor [S1, unofficial].

- **Optical Depth**:
  The optical depth is computed based on the density falloff of Rayleigh, Mie, and Ozone layers. The specific `optical_depth` function implementation is not fully detailed in the provided notes, but it integrates density along the ray path [S1, unofficial].

- **Transmittance**:
  $$T = \exp\left(-\sum_{k \in \{R, M, O\}} \sigma_k \cdot OD_k\right)$$
  where $\sigma_k$ are the scattering/absorption coefficients and $OD_k$ are the optical depths for Rayleigh (R), Mie (M), and Ozone (O) [S1, unofficial].

**Parameters**
- `POINTS_FROM_CAMERA`: Number of sample points along the view ray. Not given in the sources. [G] (Typical values in similar implementations range from 16 to 64, but must be defined by the implementer).
- `sunPosition`: Position of the sun. Provided as uniform. [S1, unofficial].
- `cameraPosition`: Position of the camera. Provided as uniform. [S1, unofficial].
- `planetPosition`: Position of the planet center. Provided as uniform. [S1, unofficial].

**Edge Cases and Pitfalls**
- **No Intersection**: If the ray does not intersect the atmosphere sphere, return the original color immediately [S1, unofficial].
- **Negative Impact Point**: Clamp the impact point to $\max(0.0, t_{in})$ [S1, unofficial].
- **Negative Distance**: Clamp the distance through atmosphere to $\max(0.0, t_{out} - t_{in})$ [S1, unofficial].
- **Non-Spherical Planets**: The clamping logic for `maximumDistance` using $t_0 - 1.0$ should not be used if the planet has landmasses or imperfect discretized geometry, as it may incorrectly truncate the ray [S1, unofficial].
- **Precision Artifacts**: Use a tolerance of $-1.0$ when clamping `maximumDistance` to $t_0$ to avoid precision artifacts from imperfect sphere discretization [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of sample points `POINTS_FROM_CAMERA` per pixel. The complexity is $O(N \cdot P)$ where $N$ is the number of pixels and $P$ is the number of samples per ray.
- **Memory Complexity**: $O(1)$ additional memory per pixel, excluding the input and output textures.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Performance scales linearly with resolution and sample count.
- **Strengths**: Post-process approach allows for easy integration into existing rendering pipelines without modifying scene geometry. Accounts for occlusion by scene geometry via depth buffer.
- **Limitations**: Limited to screen-space; rays are only traced for visible pixels. Approximates atmospheric scattering with a fixed number of samples. Relies on a spherical atmosphere model.
- **When to Use**: When rendering a spherical planet with an atmosphere and occlusion by terrain is required, and performance constraints allow for per-pixel ray marching.
- **When Not to Use**: For non-spherical planets, or when high-fidelity multi-scattering is required (this method uses single scattering approximation).

## Background
Originated from the GitHub repository `BarthPaleologue/volumetric-atmospheric-scattering` [S1, unofficial]. It is an implementation of atmospheric scattering using post-processing techniques, likely inspired by standard volumetric rendering principles and Rayleigh/Mie scattering theory.

## Sources
[S1] BarthPaleologue/volumetric-atmospheric-scattering, https://github.com/BarthPaleologue/volumetric-atmospheric-scattering, fetched 2026-10-27 (unofficial)
