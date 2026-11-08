# Atmospheric Ray Marching

Also called: Basic Volumetric Rendering Atmosphere

## Overview
Atmospheric Ray Marching is a real-time technique for rendering planetary atmospheres using single scattering, exponential density falloff, and the Rayleigh phase function [S1, unofficial]. It solves the problem of visualizing the sky and horizon glow around a spherical planet by integrating the Radiative Transfer Equation along view rays [S1, unofficial]. It sets itself apart from [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md) by specifically modeling the spherical geometry of a planet and the interaction of sunlight with atmospheric layers, and from [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) by focusing on gaseous scattering rather than volumetric cloud structures.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position, view/projection matrices, field of view (FOV), sun direction, planet radius ($R_{planet}$), atmosphere radius ($R_{atm}$), and scattering coefficients.
*   **Outputs:** Final pixel color (radiance).

**Data Structures:**
*   **Uniforms:**
    *   `cameraMatrices` (std140): Contains `viewMatrix`, `projectionMatrix`, `cameraMatrix` [S1, unofficial].
    *   `cameraUniforms` (std140): Contains `cameraPosition` (vec3), `cameraExposure` (float), `cameraFOV` (float) [S1, unofficial].
*   **Constants:**
    *   `rayleigh`: vec3(5.802, 13.558, 33.1) * 1e-6 [S1, unofficial].
    *   `ozone`: vec3(0.650, 1.881, 0.085) * 3e-6 [S1, unofficial].
    *   `sigmaS` (scattering coefficient): Equal to `rayleigh` [S1, unofficial].
    *   `sigmaT` (extinction coefficient): Equal to `rayleigh` + `ozone` [S1, unofficial].
    *   `STEPS_PRIMARY`: 32 [S1, unofficial].
    *   `STEPS_LIGHT`: 32 [S1, unofficial].
    *   `GAMMA`: 2.2 [S1, unofficial].

**Formulas:**
*   **Ray Direction:** $dir = normalize((gl\_FragCoord.xy - 0.5 \cdot resolution, -z))$, where $z = (0.5 \cdot resolution.y) / \tan(0.5 \cdot FOV)$ [S1, unofficial].
*   **Sphere Intersection:** For a sphere of radius $R$, solve $at^2 + bt + c = 0$ with $a=1$, $b=2\vec{o}\cdot\vec{d}$, $c=\vec{o}\cdot\vec{o}-R^2$. Roots are $\frac{-b \pm \sqrt{b^2-4ac}}{2a}$ [S1, unofficial].
*   **Density Falloff:** $\rho(h) = e^{-h / H}$, where $h$ is height above planet surface and $H$ is scale height [S1, unofficial].
*   **Rayleigh Phase Function:** $P(\theta) = \frac{3}{16\pi} (1 + \cos^2\theta)$, where $\theta$ is the scattering angle [S1, unofficial].
*   **Transmittance Update:** $T_{total} \leftarrow T_{total} \cdot e^{-\sigma_T \rho \Delta s}$ [S1, unofficial].
*   **Inscatter Calculation:** $L_{in} = S_{sun} C_{sun} e^{-\sigma_T \tau_{light}}$, where $\tau_{light}$ is the optical depth along the light ray [S1, unofficial].
*   **Radiance Accumulation:** $L_{out} \leftarrow L_{out} + T_{total} \cdot \sigma_S \cdot \rho \cdot P(\theta) \cdot L_{in} \cdot \Delta s$ [S1, unofficial].

**Steps:**
1.  **Ray Setup:** Calculate the view ray direction $dir$ from fragment coordinates and camera FOV [S1, unofficial].
2.  **Intersection:** Intersect the ray with the atmosphere sphere (radius $R_{atm}$) and planet sphere (radius $R_{planet}$) using the quadratic formula to determine entry and exit points [S1, unofficial].
3.  **Marching Loop:** Iterate $STEPS\_PRIMARY$ times along the view ray segment within the atmosphere:
    a.  **Sample Density:** Calculate height $h$ of the current sample point above the planet surface and compute density $\rho(h)$ [S1, unofficial].
    b.  **Shadow Test:** Intersect a light ray from the sample point towards the sun with a sphere of radius $0.95 R_{planet}$ to determine if the point is in shadow [S1, unofficial].
    c.  **Inscatter:** If not in shadow, calculate inscattered light $L_{in}$ by marching $STEPS\_LIGHT$ steps towards the sun, summing exponential density to find optical depth $\tau_{light}$, and applying attenuation [S1, unofficial].
    d.  **Transmittance:** Update total transmittance $T_{total}$ using the Beer-Lambert law with $\sigma_T$ and $\rho$ [S1, unofficial].
    e.  **Accumulate:** Add scattered radiance to $L_{out}$ using $\sigma_S$, $\rho$, $P(\theta)$, $L_{in}$, and $T_{total}$ [S1, unofficial].
4.  **Post-Processing:** Apply gamma correction with value `GAMMA` (typically 2.2) to the final color [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_atmosphere(frag_coord, resolution, camera_pos, view_matrix, proj_matrix, fov, sun_dir, sun_color, planet_radius, atm_radius, scale_height):
    # 1. Ray Setup
    z = (0.5 * resolution.y) / tan(0.5 * fov)
    dir = normalize(vec3(frag_coord.x - 0.5 * resolution.x, frag_coord.y - 0.5 * resolution.y, -z))
    
    # Transform to world space (assuming view_matrix inverse is available or dir is already in world space)
    # Note: The source implies dir is calculated in camera space or similar, 
    # but intersection requires world space. Assuming standard pipeline:
    # dir_world = inverse(view_matrix) * dir (homogeneous)
    # For simplicity, assuming dir is already in world space or camera_pos is origin for ray
    ray_o = camera_pos
    ray_d = dir
    
    # 2. Intersection
    # Intersect with Atmosphere Sphere
    a = 1.0
    b = 2.0 * dot(ray_o, ray_d)
    c = dot(ray_o, ray_o) - atm_radius * atm_radius
    disc = b*b - 4*a*c
    if disc < 0: return vec3(0.0) # No intersection
    
    sqrt_disc = sqrt(disc)
    t_atm1 = (-b - sqrt_disc) / (2.0 * a)
    t_atm2 = (-b + sqrt_disc) / (2.0 * a)
    
    # Intersect with Planet Sphere
    c_planet = dot(ray_o, ray_o) - planet_radius * planet_radius
    disc_planet = b*b - 4*a*c_planet
    
    t_planet1 = 0.0
    t_planet2 = 0.0
    hit_planet = False
    
    if disc_planet >= 0:
        sqrt_disc_p = sqrt(disc_planet)
        t_planet1 = (-b - sqrt_disc_p) / (2.0 * a)
        t_planet2 = (-b + sqrt_disc_p) / (2.0 * a)
        hit_planet = True
        
    # Determine valid segment
    # If hitting planet, march from atm entry to planet entry (or surface if inside)
    # Simplified logic based on typical atmospheric rendering:
    # Start at max(0, t_atm1) if ray originates outside, or t_atm1 if inside
    # End at min(t_atm2, t_planet1) if hitting planet, else t_atm2
    
    t_start = max(0.0, t_atm1)
    if hit_planet:
        t_end = min(t_atm2, t_planet1)
    else:
        t_end = t_atm2
        
    if t_start >= t_end: return vec3(0.0)
    
    # 3. Marching Loop
    L_out = vec3(0.0)
    T_total = 1.0
    step_size = (t_end - t_start) / STEPS_PRIMARY
    
    for i in range(STEPS_PRIMARY):
        t = t_start + i * step_size
        pos = ray_o + t * ray_d
        h = length(pos) - planet_radius # Height above surface
        rho = exp(-h / scale_height)
        
        # Shadow Test
        # Ray from pos towards sun
        light_ray_o = pos
        light_ray_d = -sun_dir # Towards sun
        
        # Intersect with shadow sphere (0.95 * planet_radius)
        r_shadow = 0.95 * planet_radius
        a_l = 1.0
        b_l = 2.0 * dot(light_ray_o, light_ray_d)
        c_l = dot(light_ray_o, light_ray_o) - r_shadow * r_shadow
        disc_l = b_l*b_l - 4*a_l*c_l
        
        in_shadow = False
        if disc_l >= 0:
            sqrt_disc_l = sqrt(disc_l)
            t_l1 = (-b_l - sqrt_disc_l) / (2.0 * a_l)
            t_l2 = (-b_l + sqrt_disc_l) / (2.0 * a_l)
            # If any root is positive, it hits the shadow sphere
            if t_l1 > 0 or t_l2 > 0:
                in_shadow = True
        
        # Inscatter Calculation
        L_in = vec3(0.0)
        if not in_shadow:
            # March light ray to calculate optical depth
            tau_light = 0.0
            light_step_size = 100.0 / STEPS_LIGHT # Arbitrary large distance, or until out of atmosphere
            # Simplified: march STEPS_LIGHT steps towards sun
            for j in range(STEPS_LIGHT):
                light_t = j * light_step_size
                light_pos = light_ray_o + light_t * light_ray_d
                light_h = length(light_pos) - planet_radius
                light_rho = exp(-light_h / scale_height)
                tau_light += light_rho * light_step_size
            
            # Attenuate sun color
            # Note: Source uses S_sun * C_sun * exp(-sigma_T * tau_light)
            # Assuming S_sun * C_sun is combined into sun_color
            L_in = sun_color * exp(-sigma_T * tau_light)
            
        # Transmittance Update
        T_total = T_total * exp(-sigma_T * rho * step_size)
        
        # Radiance Accumulation
        # Calculate scattering angle theta
        # View direction is ray_d, Light direction is -sun_dir (from sample to sun)
        # cos(theta) = dot(ray_d, -sun_dir)
        cos_theta = dot(ray_d, -sun_dir)
        P_theta = (3.0 / (16.0 * 3.14159265)) * (1.0 + cos_theta * cos_theta)
        
        L_out = L_out + T_total * sigma_S * rho * P_theta * L_in * step_size
        
    # 4. Post-Processing
    # Gamma correction
    L_out = pow(L_out, vec3(1.0 / GAMMA))
    
    return L_out
```

**Edge Cases and Pitfalls:**
*   **Shadow Banding:** Using the actual planet radius ($1.0 R_{planet}$) for shadow testing causes banding artifacts at sunset. Using a reduced radius ($0.95 R_{planet}$) avoids this [S1, unofficial].
*   **Precision:** Ensure floating-point precision is sufficient for small density values at high altitudes to prevent noise or incorrect transmittance [G].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ per pixel, where $N$ is `STEPS_PRIMARY` and $M$ is `STEPS_LIGHT`. With default values (32 steps each), this is constant per pixel but expensive for high-resolution screens.
*   **Memory Complexity:** $O(1)$ per pixel, as it uses procedural density and no textures [S1, unofficial].
*   **Performance:** Suitable for real-time rendering on modern GPUs due to the simplicity of the exponential density function and lack of texture lookups [S1, unofficial].
*   **Strengths:** Physically plausible single scattering, efficient implementation, no memory overhead for volume data [S1, unofficial].
*   **Limitations:** Single scattering approximation ignores multiple scattering effects, which can lead to dark shadows and lack of detail in dense atmospheres [S1, unofficial].

## Background
This method originates from the "Basic Volumetric Rendering" workshop at the Graphics Programming Conference in Breda, 2024 [S1, unofficial]. It is implemented as a WebGL2 application, demonstrating real-time atmospheric rendering techniques [S1, unofficial].

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
