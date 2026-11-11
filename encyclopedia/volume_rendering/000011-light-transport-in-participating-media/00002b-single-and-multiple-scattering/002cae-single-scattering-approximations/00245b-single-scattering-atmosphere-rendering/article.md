# Single-Scattering Atmosphere Rendering

Also called: Nishita Physical Sky Model, Rayleigh-Mie Single Scattering, Single Scattering Atmospheric Model, Single-Scattering Sky Rendering

## Overview
This method renders the sky and atmospheric scattering by numerically integrating single scattering along a ray through a spherical atmosphere shell, accumulating Rayleigh and Mie contributions weighted by view and light transmittance [S1, unofficial][S2, unofficial]. It solves the Radiative Transfer Equation (RTE) for a single distant light source (the sun), accounting for Rayleigh scattering, Mie scattering, and optionally Ozone absorption [S6, unofficial]. It sets itself apart from generic linear parameterizations by using non-linear sampling distributions to better distribute density near the camera [S1, unofficial], and from full multiple scattering models by ignoring indirect light bounces, making it suitable for real-time sky rendering [S6, unofficial].

## How it works
**Inputs:** Camera position $\mathbf{o}$, view direction $\boldsymbol{\omega}$, sun direction $\boldsymbol{\omega}_{sun}$, planet radius $R_p$, atmosphere radius $R_a$, integration step count $N$, and physical constants (scale heights, scattering coefficients).
**Outputs:** Scattered radiance $L_{scat}$ and transmittance $T_{view}$.

**Data Structures:**
*   `sky_radiance_and_transmittance`: Struct containing `radiance` (float3) and `transmittance` (float3).

**Physical Constants & Parameters:**
*   Rayleigh scale height $H_R = 8000$ m [S6, unofficial].
*   Mie scale height $H_M = 1200$ m [S6, unofficial].
*   Rayleigh scattering coefficient at sea level $\sigma_{s,R} = (5.8, 13.5, 33.1) \times 10^{-6} \text{ m}^{-1}$ [S6, unofficial].
*   Mie scattering coefficient at sea level $\sigma_{s,M} = 21 \times 10^{-6} \text{ m}^{-1}$ [S6, unofficial].
*   Ozone absorption coefficient at sea level $\sigma_{a,O} = (3.426, 8.298, 0.356) \times 0.06 \times 10^{-5} \text{ m}^{-1}$ [S6, unofficial].
*   Ozone scale height $H_O = 8000$ m [S6, unofficial].
*   Mie asymmetry factor $g = 0.76$ [S6, unofficial].

**Step 1: Ray-Atmosphere Intersection**
Compute the entry ($t_{in}$) and exit ($t_{out}$) distances of the view ray $\mathbf{r}(t) = \mathbf{o} + t\boldsymbol{\omega}$ intersecting the atmosphere sphere of radius $R_a$.
If the ray originates inside the planet ($t_{in} < 0$), advance the start point to the planet exit and clamp $t_{out}$ to the planet intersection if the ray hits the planet [S1, unofficial][S2, unofficial].
If the ray misses the atmosphere ($t_{in} > t_{out}$), return zero radiance [S1, unofficial].

**Step 2: Sampling Distribution**
To handle steep density gradients, use a power-law distribution for sample positions $t_i$ along the ray length $L = t_{out} - t_{in}$ [S1, unofficial]:
$$ t_i = \left(\frac{i}{N}\right)^E \cdot L, \quad i = 0 \dots N-1 $$
where the exponent $E$ depends on the ray height $h$ above the planet surface:
$$ E = 1 + \max\left(0, 1 - \frac{h}{H_{mod}}\right) \cdot 8 $$
$H_{mod}$ is a modulation height (e.g., atmosphere thickness) [S1, unofficial].

**Step 3: Integration Loop**
Initialize accumulators: `optical_depth` = 0, `rayleigh` = 0, `mie` = 0.
For each step $i$ from 0 to $N-1$:
1.  **Position:** $\mathbf{p} = \mathbf{o} + t_i \boldsymbol{\omega}$.
2.  **Height:** $h = \|\mathbf{p}\| - R_p$.
3.  **Density:** Calculate exponential density profiles [S6, unofficial]:
    $$ \rho_R(h) = \exp\left(-\frac{\max(0, h)}{H_R}\right) $$
    $$ \rho_M(h) = \exp\left(-\frac{\max(0, h)}{H_M}\right) $$
    $$ \rho_O(h) = \exp\left(-\frac{\max(0, h)}{H_O}\right) $$
4.  **Coefficients:**
    $$ \sigma_R = \sigma_{s,R} \cdot \rho_R(h) $$
    $$ \sigma_M = \sigma_{s,M} \cdot \rho_M(h) $$
    $$ \sigma_O = \sigma_{a,O} \cdot \rho_O(h) $$
5.  **Optical Depth Step:**
    $$ \Delta \tau = (\sigma_R + 1.11 \sigma_M + \sigma_O) \cdot \Delta t $$
    where $\Delta t = t_i - t_{i-1}$ (with $t_{-1}=0$) [S6, unofficial].
    Update `optical_depth` += $\Delta \tau$.
6.  **Transmittances:**
    *   View transmittance from camera to $\mathbf{p}$: $T_{view} = \exp(-\text{optical\_depth})$.
    *   Light transmittance from $\mathbf{p}$ to sun: Compute the optical depth along the sun direction from $\mathbf{p}$ to the atmosphere exit $t_{sun\_out}$ (intersection of ray from $\mathbf{p}$ in direction $\boldsymbol{\omega}_{sun}$ with $R_a$). Sum the extinction coefficients along this path to get $\tau_{light}$. Then $T_{light} = \exp(-\tau_{light})$.
7.  **Phase Functions:**
    *   Cosine of scattering angle: $\mu = \boldsymbol{\omega} \cdot \boldsymbol{\omega}_{sun}$.
    *   Rayleigh phase: $P_R(\mu) = \frac{3}{16\pi}(1 + \mu^2)$ [S6, unofficial].
    *   Mie phase (Henyey-Greenstein): $P_M(\mu) = \frac{1-g^2}{4\pi(1+g^2-2g\mu)^{1.5}}$ [S6, unofficial].
8.  **Accumulation:**
    $$ \text{rayleigh} += T_{view} \cdot T_{light} \cdot P_R(\mu) \cdot \sigma_R \cdot \Delta t $$
    $$ \text{mie} += T_{view} \cdot T_{light} \cdot P_M(\mu) \cdot \sigma_M \cdot \Delta t $$

**Step 4: Final Radiance**
$$ L_{scat} = (\text{rayleigh} \cdot \mathbf{C}_{sun} + \text{mie} \cdot \mathbf{C}_{sun}) $$
where $\mathbf{C}_{sun}$ is the sun color/intensity [S1, unofficial].
Final transmittance: $T_{final} = \exp(-\text{optical\_depth})$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scattering_atmosphere(ray_origin, ray_dir, sun_dir, planet_radius, atmosphere_radius, step_count):
    # 1. Intersect with atmosphere
    t_in, t_out = ray_sphere_intersection(ray_origin, ray_dir, atmosphere_radius)
    
    # Handle ray starting inside planet
    if t_in < 0:
        # Advance to planet surface if hitting planet, else clamp to 0
        t_planet_in, t_planet_out = ray_sphere_intersection(ray_origin, ray_dir, planet_radius)
        if t_planet_out > 0:
            t_in = max(t_in, t_planet_out) # Start from planet exit
        else:
            t_in = 0
            
    if t_in >= t_out:
        return 0.0, 1.0 # Missed atmosphere or inside planet without exit

    ray_length = t_out - t_in
    current_t = t_in
    
    # 2. Setup constants
    Hr = 8000.0
    Hm = 1200.0
    Ho = 8000.0
    sigma_s_R = vec3(5.8, 13.5, 33.1) * 1e-6
    sigma_s_M = vec3(21.0, 21.0, 21.0) * 1e-6
    sigma_a_O = vec3(3.426, 8.298, 0.356) * 0.06 * 1e-5
    g = 0.76
    pi = 3.14159265359
    
    # Modulation height for sampling exponent (e.g., atmosphere thickness)
    H_mod = atmosphere_radius - planet_radius
    
    # 3. Integration
    optical_depth = 0.0
    rayleigh_acc = vec3(0.0)
    mie_acc = vec3(0.0)
    
    # Pre-calculate sun intersection from origin to get light path length? 
    # No, light transmittance is per sample point.
    
    for i in range(step_count):
        # Power-law sampling
        exponent = 1.0 + max(0.0, 1.0 - (length(ray_origin + current_t * ray_dir) - planet_radius) / H_mod) * 8.0
        t_sample = t_in + pow(float(i) / float(step_count), exponent) * ray_length
        
        # Step size
        dt = t_sample - current_t
        current_t = t_sample
        
        # Position and Height
        p = ray_origin + current_t * ray_dir
        h = length(p) - planet_radius
        
        # Densities
        rho_R = exp(-max(0.0, h) / Hr)
        rho_M = exp(-max(0.0, h) / Hm)
        rho_O = exp(-max(0.0, h) / Ho)
        
        # Coefficients
        s_R = sigma_s_R * rho_R
        s_M = sigma_s_M * rho_M
        s_O = sigma_a_O * rho_O
        
        # Optical depth step
        sigma_t = s_R + 1.11 * s_M + s_O
        optical_depth += dot(sigma_t, vec3(1.0)) * dt # Assuming scalar OD for T, but T is vec3 exp(-OD)
        # Note: Transmittance is component-wise exp(-OD)
        
        # View Transmittance
        T_view = exp(-optical_depth) # Component-wise
        
        # Light Transmittance
        # Intersect ray from p in sun_dir with atmosphere
        t_sun_in, t_sun_out = ray_sphere_intersection(p, sun_dir, atmosphere_radius)
        # If sun is behind, t_sun_in might be negative, but we only care about path out
        # If ray misses, T_light = 1 (no attenuation) or 0? Usually 1 if no atmosphere in path.
        if t_sun_out > 0:
            # Integrate optical depth from p to t_sun_out
            # Simplified: assume constant sigma_t over small step? No, need integration.
            # For implementation, use a fixed number of steps for light path or analytical approximation.
            # Here we use a simplified single-step approximation for the light path from p to exit
            # Or use the same density profile logic.
            # Let's use a helper function for light OD
            tau_light = integrate_optical_depth(p, sun_dir, t_sun_out, planet_radius, Hr, Hm, Ho, sigma_s_R, sigma_s_M, sigma_a_O)
            T_light = exp(-tau_light)
        else:
            T_light = vec3(1.0)
            
        # Phase Functions
        mu = dot(ray_dir, sun_dir)
        P_R = (3.0 / (16.0 * pi)) * (1.0 + mu * mu)
        denom = 1.0 + g*g - 2.0*g*mu
        P_M = (1.0 - g*g) / (4.0 * pi * pow(denom, 1.5))
        
        # Accumulate
        rayleigh_acc += T_view * T_light * P_R * s_R * dt
        mie_acc += T_view * T_light * P_M * s_M * dt
        
    # Final Radiance
    # Sun intensity/color is applied here
    return rayleigh_acc + mie_acc, exp(-optical_depth)

def integrate_optical_depth(start_pos, dir, max_t, planet_radius, Hr, Hm, Ho, sigma_s_R, sigma_s_M, sigma_a_O):
    # Simplified integration: sample midpoints
    steps = 8
    ds = max_t / steps
    tau = vec3(0.0)
    for i in range(steps):
        s = start_pos + (i + 0.5) * ds * dir
        h = length(s) - planet_radius
        rho_R = exp(-max(0.0, h) / Hr)
        rho_M = exp(-max(0.0, h) / Hm)
        rho_O = exp(-max(0.0, h) / Ho)
        sigma_t = sigma_s_R * rho_R + 1.11 * sigma_s_M * rho_M + sigma_a_O * rho_O
        tau += sigma_t * ds
    return tau
```

## Subtopics

### [Sky View LUT with Angular Compression](6648-sky-view-lut-with-angular-compression.md)
A technique for rendering sky and atmosphere using a precomputed Look-Up Table (LUT) with angular compression, ported from a reference implementation. It sets itself apart from generic linear parameterizations by using non-linear mappings for altitude and angles to better distribute sampling density according to the physical behavior of light scattering in spherical atmospheres, building on [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md).

### [Spectral Scattering Model for Atmosphere](9399-spectral-scattering-model-for-atmosphere.md)
This method provides a physically-based, fully spectral real-time rendering model for planetary atmospheres and large-scale water bodies, accounting for direct illumination, indirect scattered light, interreflections between surface and volume, and seabed reflections. It distinguishes itself from its siblings by utilizing fully spectral computations (rather than RGB) and supporting arbitrarily dense atmospheres and water bodies within a single precomputation framework, whereas [Sky View LUT with Angular Compression](6648-sky-view-lut-with-angular-compression.md) focuses on angular compression for sky views without the coupled water body simulation.

### Water Scattering Precomputation
Uses different physical constants and parameterization for the water texture compared to atmospheric scattering [S3]. Incident light for water is attenuated through the atmosphere: $I_0^I(\lambda) = I^I(\lambda)e^{-t(P_g P, \lambda)}$ [S3]. The 4D water scattering texture is reduced to a 3D texture for rendering by using only the subset corresponding to observer depth $d=0$m [S3]. If observers below the water surface are needed, the full 4D texture can be used [S3]. For low accuracy requirements, only the first scattering order is needed because water is a weakly scattering medium [S3].

### Water Scattering Model
Similar to atmospheric scattering but the volume is between water level and seabed. Water has significant absorption (mainly red) and constant density, so the density scale function $\rho$ is removed [S3]. Attenuation coefficient: $t^{\text{water}}(S, \lambda) = \|S\| (\zeta^a + \zeta^b)(\lambda)$ [S3]. Mie scattering is not explicitly modeled in water because data for seawater is unavailable and it is strongly anisotropic [S3].

### Jackèl and Walter Atmospheric Rendering
Highly realistic atmospheric rendering, allowing modeling of real world atmospheres (composed by ozone, haze, dust, soot, sulphur acid, etc.) [S8]. Combines four sub-models (clear air, aerosol, ozone, rain layer) to determine atmospheric model [S8]. Accounts for both Rayleigh and Mie scattering and approximates second order scattering [S8]. Authors do not explain how the second order scattering approximation is performed [S8].

### Preetham Sky Model
Considers the rendering of outdoor scenes including sunlight and skylight, and the effects of aerial perspective [S8]. Computes the sky spectral radiance function for a set of sun positions and turbidities (using the method of Nishita et al. [59]), then fits a parametric function [S8]. Wilkie et al. propose a simple analytical model for skylight polarisation to be added to Preetham’s skylight model [S8].

### Jensen Nighttime Sky Rendering
Rendering of nighttime sky including illumination from the Moon, stars, zodiacal light, and atmosphere [S8]. Uses the atmospheric model and scattering phase functions of Nishita [59] [S8]. Simulates transport with multiple scattering combining distribution ray-tracing and ray-marching [S8].

## Operating characteristics
*   **Time Complexity:** $O(STEPS)$ per pixel for the luminance integral and $O(STEPS)$ for transmittance calculations, where STEPS is the number of integration segments [S6, unofficial].
*   **Memory Complexity:** $O(1)$ if computed per-pixel; $O(N)$ if precomputed into a LUT.
*   **Performance:** Real-time capable with low step counts (e.g., 18-64 steps) [S1, unofficial][S2, unofficial].
*   **Strengths:** Physically plausible sky colors, handles sun position dynamically, efficient for single scattering.
*   **Limitations:** Ignores multiple scattering, which causes inaccuracies in deep atmospheres or high optical depths [S6, unofficial]. Approximations in light transmittance integration can introduce errors.
*   **When to use:** Real-time sky rendering, planetary visualization.
*   **When not to use:** Scenes requiring accurate deep atmospheric scattering or volumetric fog within the atmosphere.

## Background
Originated from Nishita et al.'s physical sky model [S3][S8]. Used in real-time sky rendering, specifically referenced in Sébastien Hillaire's 2016 course and 2020 paper, and Scratch a Pixel tutorial [S6, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S2] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S3] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
[S4] Depersonalizc/volumetric-clouds, https://github.com/Depersonalizc/volumetric-clouds, fetched 2026-10-27 (unofficial)
[S5] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
[S6] Sky and Atmosphere rendering: A physical approach, https://cpp-rendering.io/sky-and-atmosphere-rendering/, fetched 2026-10-28 (unofficial)
[S7] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S8] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
