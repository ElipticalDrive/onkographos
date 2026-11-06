# Atmospheric Ray Marching

Also called: Basic Volumetric Rendering Atmosphere

## Overview
Atmospheric Ray Marching renders planetary atmospheres using single scattering, exponential density falloff, and the Rayleigh phase function [S1, unofficial]. It solves the problem of visualizing the color and brightness of a sky dome by integrating light along view rays through a spherical medium. It is distinguished from general volumetric rendering by its specific analytic density model for a planet and its focus on Rayleigh scattering physics [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position ($ro$), view direction ($v$), sun direction ($L_{sun}$), planet radius ($R_{planet}$), atmosphere radius ($R_{atm}$), and physical coefficients (scattering, absorption).
*   **Outputs:** Final pixel color ($C_{out}$).

**Data Structures:**
*   **Uniforms:**
    *   `cameraMatrices` (std140): Contains `viewMatrix`, `projectionMatrix`, `cameraMatrix` [S1, unofficial].
    *   `cameraUniforms` (std140): Contains `cameraPosition` (vec3), `cameraExposure` (float), `cameraFOV` (float) [S1, unofficial].
*   **Constants:**
    *   `rayleigh`: vec3(5.802, 13.558, 33.1) * 1e-6 [S1, unofficial].
    *   `ozone`: vec3(0.650, 1.881, 0.085) * 3e-6 [S1, unofficial].
    *   `sigmaS` (Scattering coefficient): Equal to `rayleigh` [S1, unofficial].
    *   `sigmaT` (Extinction coefficient): Equal to `rayleigh` + `ozone` [S1, unofficial].
    *   `STEPS_PRIMARY`: 32 [S1, unofficial].
    *   `STEPS_LIGHT`: 32 [S1, unofficial].
    *   `GAMMA`: 2.2 [S1, unofficial].

**Formulas:**
*   **Exponential Density Falloff:** $\rho(h) = e^{-h / H}$, where $h$ is height above planet surface and $H$ is scale height [S1, unofficial].
*   **Rayleigh Phase Function:** $P(\theta) = \frac{3}{16\pi} (1 + \cos^2\theta)$, where $\theta$ is the scattering angle between view direction and sun direction [S1, unofficial].
*   **Ray-Sphere Intersection:** For ray origin $o$ and direction $d$ against sphere radius $R$: $a=1, b=2(o \cdot d), c=(o \cdot o) - R^2$. Roots are $\frac{-b \pm \sqrt{b^2-4ac}}{2a}$ [S1, unofficial].

**Algorithm Steps:**
1.  **Ray Generation:** Calculate view direction $v$ from fragment coordinates and camera FOV: $v = \text{normalize}((gl\_FragCoord.xy - 0.5 \times resolution, -z))$, where $z = (0.5 \times resolution.y) / \tan(0.5 \times FOV)$ [S1, unofficial].
2.  **Intersection:** Intersect the view ray with the atmosphere sphere ($R_{atm}$) and planet sphere ($R_{planet}$) to determine entry and exit points [S1, unofficial].
3.  **Initialization:** Initialize transmittance $T_{total} = 1.0$ and accumulated radiance $L_{out} = 0$ [S1, unofficial].
4.  **Marching Loop:** Iterate $i$ from 0 to $STEPS\_PRIMARY - 1$:
    a.  **Sample Point:** Calculate current position $p$ along the view ray [S1, unofficial].
    b.  **Density:** Calculate local density $\rho$ using height $h$ of $p$ relative to planet surface [S1, unofficial].
    c.  **Shadow Test:** Determine if $p$ is in shadow by casting a light ray towards the sun. Intersect this light ray with a sphere of radius $0.95 R_{planet}$ to avoid banding artifacts [S1, unofficial].
    d.  **Inscatter:** If not in shadow, calculate inscattered light $L_{in} = S_{sun} C_{sun} e^{-\sigma_T \tau_{light}}$, where $\tau_{light}$ is the optical depth from $p$ to the sun [S1, unofficial].
    e.  **Optical Depth (Light):** Calculate $\tau_{light}$ by marching $STEPS\_LIGHT$ steps towards the sun, summing $\rho \cdot \Delta s$ [S1, unofficial].
    f.  **Transmittance Update:** Update total transmittance: $T_{total} \leftarrow T_{total} \cdot e^{-\sigma_T \rho \Delta s}$ [S1, unofficial].
    g.  **Accumulation:** Add scattered light to output: $L_{out} \leftarrow L_{out} + T_{total} \cdot \sigma_S \cdot \rho \cdot P(\theta) \cdot L_{in} \cdot \Delta s$ [S1, unofficial].
5.  **Final Output:** Apply gamma correction to $L_{out}$ and return [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atmospheric_ray_marching(ro, v, sun_dir, R_planet, R_atm, H, sigma_S, sigma_T, S_sun, C_sun, STEPS_PRIMARY, STEPS_LIGHT):
    # Intersect view ray with atmosphere and planet
    t_atm = ray_sphere_intersect(ro, v, R_atm)
    t_planet = ray_sphere_intersect(ro, v, R_planet)
    
    # Determine start and end points for marching
    # Assuming ro is outside atmosphere
    t_start = t_atm
    t_end = t_planet if t_planet > 0 else t_atm # Simplified logic for atmosphere-only or planet-intersecting
    
    if t_start >= t_end:
        return vec3(0.0) # Missed atmosphere

    L_out = vec3(0.0)
    T_total = 1.0
    step_size = (t_end - t_start) / STEPS_PRIMARY

    for i in range(STEPS_PRIMARY):
        t = t_start + i * step_size
        p = ro + v * t
        h = length(p) - R_planet # Height above surface
        
        # Exponential density falloff
        rho = exp(-h / H)
        
        # Shadow test: Ray from p towards sun
        # Use 0.95 * R_planet to avoid banding artifacts
        shadow_t = ray_sphere_intersect(p, sun_dir, 0.95 * R_planet)
        in_shadow = shadow_t > 0
        
        L_in = vec3(0.0)
        if not in_shadow:
            # Calculate optical depth to sun
            tau_light = 0.0
            light_step = 0.1 # Arbitrary small step, or normalized
            # March STEPS_LIGHT towards sun
            for j in range(STEPS_LIGHT):
                p_light = p + sun_dir * (j * light_step)
                h_light = length(p_light) - R_planet
                rho_light = exp(-h_light / H)
                tau_light += rho_light * light_step
            
            # Inscatter calculation
            # S_sun is sun intensity, C_sun is sun color
            L_in = S_sun * C_sun * exp(-sigma_T * tau_light)
        
        # Update transmittance
        T_total *= exp(-sigma_T * rho * step_size)
        
        # Phase function
        cos_theta = dot(normalize(v), -sun_dir) # Angle between view and light
        P_theta = (3.0 / (16.0 * PI)) * (1.0 + cos_theta * cos_theta)
        
        # Accumulate radiance
        L_out += T_total * sigma_S * rho * P_theta * L_in * step_size

    # Gamma correction
    L_out = pow(L_out, vec3(1.0 / GAMMA))
    return L_out

def ray_sphere_intersect(o, d, R):
    a = 1.0
    b = 2.0 * dot(o, d)
    c = dot(o, o) - R * R
    discriminant = b * b - 4 * a * c
    
    if discriminant < 0:
        return -1.0
    
    sqrt_disc = sqrt(discriminant)
    t1 = (-b - sqrt_disc) / (2 * a)
    t2 = (-b + sqrt_disc) / (2 * a)
    
    # Return smallest positive t
    if t1 > 0:
        return t1
    elif t2 > 0:
        return t2
    else:
        return -1.0
```

## Operating characteristics
*   **Complexity:** Time complexity is $O(STEPS\_PRIMARY \times (1 + STEPS\_LIGHT))$ per pixel due to the nested light ray marching for shadow testing [S1, unofficial]. Memory complexity is $O(1)$ per pixel as it uses iterative accumulation.
*   **Performance:** The method is computationally expensive due to the dual ray marching (view and light) and exponential calculations per step [S1, unofficial].
*   **Strengths:** Physically plausible single-scattering approximation for clear skies; analytic density allows for infinite resolution without texture artifacts [S1, unofficial].
*   **Limitations:** Single scattering ignores multiple scattering events, leading to dark shadows and lack of haze in dense atmospheres [S1, unofficial]. The fixed step count may cause banding or noise if insufficient [S1, unofficial].
*   **When to use:** Real-time rendering of clear planetary atmospheres where performance is critical but physical accuracy for single scattering is desired [S1, unofficial].
*   **When not to use:** Scenes requiring multiple scattering, clouds, or complex aerosol distributions [S1, unofficial].

## Background
This method originates from the "Basic Volumetric Rendering" workshop at the Graphics Programming Conference held in Breda, 2024 [S1, unofficial]. It is implemented in the `al-ro/volumetrics` repository [S1, unofficial]. The technique is a specific application of ray marching for atmospheric rendering, focusing on Rayleigh scattering and exponential density models [S1, unofficial].

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
