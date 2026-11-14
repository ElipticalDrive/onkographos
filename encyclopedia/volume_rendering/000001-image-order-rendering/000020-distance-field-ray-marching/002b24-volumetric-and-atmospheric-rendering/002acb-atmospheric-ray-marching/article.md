# Atmospheric Ray Marching

Also called: Basic Volumetric Rendering Atmosphere

## Overview
Atmospheric Ray Marching is a technique for rendering planetary atmospheres using single scattering, exponential density falloff, and the Rayleigh phase function [S1, unofficial]. It solves the Radiative Transfer Equation (RTE) for a spherical atmosphere by marching along view rays and light rays to compute inscattered sunlight and shadowing [S1, unofficial]. It is distinguished from [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md) by its specific application to planetary spheres with analytical density profiles and its handling of shadow rays for occlusion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position $\mathbf{o}$, direction $\mathbf{d}$, planet radius $R_{planet}$, atmosphere radius $R_{atm}$, sun direction $\mathbf{s}$, and physical coefficients (scattering, absorption).
*   **Outputs:** Final pixel color $L_{out}$.

**Data Structures:**
*   **Uniforms:**
    *   `cameraMatrices` (std140): Contains `viewMatrix`, `projectionMatrix`, `cameraMatrix` [S1, unofficial].
    *   `cameraUniforms` (std140): Contains `cameraPosition` (vec3), `cameraExposure` (float), `cameraFOV` (float) [S1, unofficial].
*   **Constants:**
    *   `rayleigh`: vec3 $(5.802, 13.558, 33.1) \times 10^{-6}$ [S1, unofficial].
    *   `ozone`: vec3 $(0.650, 1.881, 0.085) \times 3 \times 10^{-6}$ [S1, unofficial].
    *   `sigmaS` (scattering coefficient): Equal to `rayleigh` [S1, unofficial].
    *   `sigmaT` (extinction coefficient): Equal to `rayleigh` + `ozone` [S1, unofficial].
    *   `STEPS_PRIMARY`: 32 [S1, unofficial].
    *   `STEPS_LIGHT`: 32 [S1, unofficial].
    *   `GAMMA`: 2.2 [S1, unofficial].

**Formulas:**
*   **Rayleigh Phase Function:** $P(\theta) = \frac{3}{16\pi} (1 + \cos^2\theta)$, where $\theta$ is the scattering angle between the view direction and the sun direction [S1, unofficial].
*   **Exponential Density Falloff:** $\rho(h) = e^{-h / H}$, where $h$ is the height above the planet surface and $H$ is the scale height [S1, unofficial].
*   **Sphere Intersection:** For a ray $r(t) = \mathbf{o} + t\mathbf{d}$ and sphere radius $R$, solve $at^2 + bt + c = 0$ with $a=1$, $b=2\mathbf{o}\cdot\mathbf{d}$, $c=\mathbf{o}\cdot\mathbf{o}-R^2$. Roots are $t = \frac{-b \pm \sqrt{b^2-4ac}}{2a}$ [S1, unofficial].

**Algorithm Steps:**
1.  **Ray Direction:** Calculate the view ray direction $\mathbf{d}$ from fragment coordinates $(x, y)$ and resolution $res$:
    $$ \mathbf{d} = \text{normalize}\left( \left( \frac{x}{res.x} - 0.5, \frac{y}{res.y} - 0.5, -z \right) \right) $$
    where $z = \frac{0.5 \cdot res.y}{\tan(0.5 \cdot FOV)}$ [S1, unofficial].
2.  **Intersect Atmosphere:** Compute the entry and exit points of the view ray with the atmosphere sphere ($R_{atm}$) and planet sphere ($R_{planet}$) using the quadratic formula [S1, unofficial]. Determine the valid segment $[t_{start}, t_{end}]$ for marching.
3.  **Initialize Accumulators:** Set total transmittance $T_{total} = 1.0$ and accumulated radiance $L_{out} = 0$ [S1, unofficial].
4.  **March View Ray:** Loop $i$ from 0 to $STEPS\_PRIMARY - 1$:
    a.  Calculate step size $\Delta s = \frac{t_{end} - t_{start}}{STEPS\_PRIMARY}$.
    b.  Compute current position $\mathbf{p} = \mathbf{o} + (t_{start} + i \cdot \Delta s) \mathbf{d}$.
    c.  Compute height $h = \|\mathbf{p}\| - R_{planet}$.
    d.  Evaluate density $\rho = e^{-h / H}$ [S1, unofficial].
    e.  **Shadow Test:** Cast a shadow ray from $\mathbf{p}$ towards the sun $\mathbf{s}$. Intersect this ray with a sphere of radius $0.95 R_{planet}$ to determine if the point is in shadow. Using $0.95 R_{planet}$ avoids banding artifacts at sunset [S1, unofficial].
    f.  **Inscattering Calculation:** If not in shadow:
        i.  Compute optical depth to the sun $\tau_{light}$ by marching $STEPS\_LIGHT$ steps towards the sun, summing $\rho \cdot \Delta s_{light}$ [S1, unofficial].
        ii. Compute inscattered radiance $L_{in} = S_{sun} C_{sun} e^{-\sigma_T \tau_{light}}$, where $S_{sun}$ is solar spectral irradiance and $C_{sun}$ is a constant [S1, unofficial].
    g.  **Update Transmittance:** $T_{total} \leftarrow T_{total} \cdot e^{-\sigma_T \rho \Delta s}$ [S1, unofficial].
    h.  **Accumulate:** $L_{out} \leftarrow L_{out} + T_{total} \cdot \sigma_S \cdot \rho \cdot P(\theta) \cdot L_{in} \cdot \Delta s$ [S1, unofficial].
5.  **Gamma Correction:** Apply gamma correction to $L_{out}$ using `GAMMA` (typically 2.2) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def atmospheric_ray_marching(camera_pos, camera_dir, sun_dir, planet_radius, atm_radius, scale_height):
    # Constants
    rayleigh = vec3(5.802, 13.558, 33.1) * 1e-6
    ozone = vec3(0.650, 1.881, 0.085) * 3e-6
    sigma_s = rayleigh
    sigma_t = rayleigh + ozone
    steps_primary = 32
    steps_light = 32
    gamma = 2.2
    
    # Ray direction from fragment coords (assuming normalized device coords -1 to 1)
    # dir = normalize(vec3(ndc_x, ndc_y, -1.0 / tan(fov/2)))
    dir = normalize(camera_dir)
    
    # Intersect with atmosphere and planet
    # Solve quadratic for atm_radius and planet_radius
    # Get t_start and t_end for the visible segment
    t_start, t_end = intersect_spheres(camera_pos, dir, planet_radius, atm_radius)
    
    if t_end < t_start:
        return vec3(0.0)
        
    delta_s = (t_end - t_start) / steps_primary
    t_total_transmittance = 1.0
    l_out = vec3(0.0)
    
    for i in range(steps_primary):
        t = t_start + i * delta_s
        p = camera_pos + dir * t
        h = length(p) - planet_radius
        rho = exp(-h / scale_height)
        
        # Shadow test
        in_shadow = is_in_shadow(p, sun_dir, planet_radius * 0.95, steps_light)
        
        l_in = vec3(0.0)
        if not in_shadow:
            tau_light = compute_optical_depth_to_sun(p, sun_dir, sigma_t, steps_light, scale_height, planet_radius)
            # S_sun and C_sun are constants representing solar irradiance
            l_in = solar_irradiance * exp(-sigma_t * tau_light)
            
        # Phase function
        cos_theta = dot(normalize(p), -sun_dir) # Approximation for scattering angle depending on geometry
        # More accurately: angle between view direction and sun direction at the sample point
        # P(theta) = 3/(16*pi) * (1 + cos^2(theta))
        # theta is angle between -dir (view) and sun_dir? 
        # Standard Rayleigh: angle between incident light and view direction.
        # Incident light comes from sun_dir. View direction is -dir (looking back).
        # So cos_theta = dot(-sun_dir, -dir) = dot(sun_dir, dir) ? 
        # Actually, scattering angle is between incoming light direction and outgoing view direction.
        # Incoming: -sun_dir. Outgoing: -dir.
        # cos_theta = dot(-sun_dir, -dir) = dot(sun_dir, dir)
        cos_theta = dot(sun_dir, dir)
        phase = (3.0 / (16.0 * 3.14159265)) * (1.0 + cos_theta * cos_theta)
        
        # Update transmittance
        t_total_transmittance *= exp(-sigma_t * rho * delta_s)
        
        # Accumulate
        l_out += t_total_transmittance * sigma_s * rho * phase * l_in * delta_s
        
    # Gamma correction
    l_out = pow(l_out, vec3(1.0 / gamma))
    
    return l_out

def is_in_shadow(point, sun_dir, shadow_radius, steps):
    # Ray from point towards sun
    ray_o = point
    ray_d = sun_dir
    # Intersect with sphere of radius shadow_radius
    # If intersection distance > 0, it's in shadow (blocked by planet)
    # Note: The source says "intersecting a light ray from sample point towards sun with a sphere... to avoid banding"
    # If the ray hits the planet sphere, it is in shadow.
    t_hit = intersect_sphere(ray_o, ray_d, shadow_radius)
    return t_hit > 0.0

def compute_optical_depth_to_sun(point, sun_dir, sigma_t, steps, scale_height, planet_radius):
    tau = 0.0
    delta_s = 1000.0 # Arbitrary large step or determined by geometry
    # March towards sun
    for i in range(steps):
        p = point + sun_dir * (i * delta_s)
        h = length(p) - planet_radius
        rho = exp(-h / scale_height)
        tau += rho * delta_s
    return tau * sigma_t
```

## Operating characteristics
*   **Complexity:** Time complexity is $O(N \cdot M)$ per pixel, where $N$ is `STEPS_PRIMARY` and $M$ is `STEPS_LIGHT` [S1, unofficial]. Memory complexity is $O(1)$ per pixel as it does not store volumetric grids.
*   **Performance:** The method is computationally expensive due to the nested ray marching (view ray and light ray) and exponential density calculations [S1, unofficial].
*   **Strengths:** Physically plausible single scattering for planetary atmospheres; simple implementation; no precomputation required [S1, unofficial].
*   **Limitations:** Single scattering approximation ignores multiple scattering, which can lead to inaccuracies in dense atmospheres or near the horizon [S1, unofficial]. Shadow banding artifacts are mitigated by using $0.95 R_{planet}$ but not eliminated entirely [S1, unofficial].
*   **When to use:** Real-time rendering of planetary atmospheres where performance is critical and single scattering is sufficient [S1, unofficial].
*   **When not to use:** Scenes requiring multiple scattering or high-fidelity cloud rendering [S1, unofficial].

## Background
The method originates from the "Basic Volumetric Rendering" workshop at the Graphics Programming Conference in Breda, 2024 [S1, unofficial]. It is based on the repository `al-ro/volumetrics` [S1, unofficial]. The technique is a specific application of ray marching for atmospheric rendering, utilizing the Rayleigh phase function and exponential density profiles [S1, unofficial].

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
