# Single-Scattering Atmosphere Rendering

Also called: Nishita Physical Sky Model, Rayleigh-Mie Single Scattering, Single Scattering Atmospheric Model, Single-Scattering Sky Rendering

## Overview
Single-Scattering Atmosphere Rendering computes the color of the sky and atmospheric haze by numerically integrating the in-scattered light from a single distant light source (the sun) along a view ray passing through a spherical atmosphere shell [S1, S6, unofficial]. It solves the Radiative Transfer Equation (RTE) under the assumption that light scatters only once before reaching the observer, accounting for Rayleigh (gas) and Mie (aerosol) scattering, as well as ozone absorption [S6, unofficial]. It sets itself apart from [Multiple Scattering Models](10687-multiple-scattering-models.md) by ignoring indirect light bounces, making it computationally efficient but inaccurate for deep atmospheres or high optical depths [S6, unofficial]. It differs from [Single Scattering Approximation](5264-single-scattering-approximation.md) by specifically modeling the spherical geometry of a planetary atmosphere with exponential density profiles [S1, S6, unofficial].

## How it works
**Inputs:**
*   Observer position $P_O$ (typically inside the atmosphere).
*   View direction $\omega$ (normalized vector from camera to fragment).
*   Sun direction $\omega_{sun}$ (normalized vector from observer to sun).
*   Planet radius $R_p$ and Atmosphere outer radius $R_a$.
*   Scattering coefficients for Rayleigh ($\sigma_s^R$), Mie ($\sigma_s^M$), and Ozone ($\sigma_a^O$) at sea level.
*   Scale heights for Rayleigh ($H_R$), Mie ($H_M$), and Ozone ($H_O$).
*   Sun luminance $L_{sun}$ or ground illuminance to derive outer space luminance.
*   Integration step count $N$ (e.g., 32–64).

**Outputs:**
*   Scattered radiance $L_{sky}$ (RGB).
*   Transmittance $T_{view}$ (RGB, optional, for ground visibility).

**Data Structures:**
*   `AtmosphereParams`: Struct containing $R_p, R_a, H_R, H_M, H_O, \sigma_s^R, \sigma_s^M, \sigma_a^O$.
*   `Ray`: Origin $O$, Direction $D$, Length $L$.

**Formulas:**
1.  **Density Profile:** The density of species $i \in \{R, M, O\}$ at height $h$ above the planet surface is exponential:
    $$ \rho_i(h) = \exp\left(-\frac{\max(0, h)}{H_i}\right) $$
    where $h = \|P\| - R_p$ [S1, S6, unofficial].

2.  **Extinction Coefficient:** The total extinction coefficient at height $h$ is:
    $$ \sigma_t(h) = \sigma_s^R(h) + 1.11 \cdot \sigma_s^M(h) + \sigma_a^O(h) $$
    The factor 1.11 scales Mie scattering to account for extinction differences [S6, unofficial].
    $$ \sigma_s^R(h) = \sigma_{s,0}^R e^{-h/H_R}, \quad \sigma_s^M(h) = \sigma_{s,0}^M e^{-h/H_M}, \quad \sigma_a^O(h) = \sigma_{a,0}^O e^{-h/H_O} $$
    Typical sea-level values: $\sigma_{s,0}^R = (5.8, 13.5, 33.1) \times 10^{-6} \, m^{-1}$, $\sigma_{s,0}^M = 21 \times 10^{-6} \, m^{-1}$, $\sigma_{a,0}^O = (3.426, 8.298, 0.356) \times 0.06 \times 10^{-5} \, m^{-1}$ [S6, unofficial]. Scale heights: $H_R=8000m, H_M=1200m, H_O=8000m$ [S6, unofficial].

3.  **Phase Functions:**
    *   Rayleigh: $P_R(\cos\theta) = \frac{3}{16\pi}(1 + \cos^2\theta)$ [S1, S6, unofficial].
    *   Mie (Henyey-Greenstein): $P_M(\cos\theta) = \frac{1-g^2}{4\pi(1+g^2-2g\cos\theta)^{1.5}}$ with asymmetry factor $g=0.76$ [S6, unofficial].
    *   $\cos\theta = \omega \cdot \omega_{sun}$ [S1, unofficial].

4.  **Transmittance:** The transmittance from point $A$ to $B$ is:
    $$ Tr(A \rightarrow B) = \exp\left(-\int_A^B \sigma_t(s) ds\right) $$
    This is computed numerically by summing $\sigma_t$ at sample midpoints [S6, unofficial].

5.  **Source Term:** The in-scattered source term $j(x)$ at position $x$ is:
    $$ j(x) = L_{outerspace} \cdot Tr(outerspace \rightarrow x) \cdot \Omega_{sun} \cdot (\sigma_s^R(x)P_R + \sigma_s^M(x)P_M) $$
    where $\Omega_{sun} \approx 0.0000664 \, sr$ [S6, unofficial]. $L_{outerspace}$ is derived from ground illuminance $E_{ground} \approx 120,000$ lux and zenith transmittance $Tr_{Zenith}$ [S6, unofficial].

**Algorithm Steps:**
1.  **Ray-Atmosphere Intersection:** Compute the entry ($t_{in}$) and exit ($t_{out}$) distances of the view ray from the atmosphere sphere. If the ray originates inside the planet, advance the start point to the planet exit [S1, S5, unofficial].
2.  **Sample Distribution:** Generate sample positions $t_i$ along the ray. Uniform spacing is common, but power-law distribution $t_i = (\frac{i}{N})^E \cdot L$ with exponent $E$ improves convergence near the camera [S1, unofficial].
3.  **Loop Integration:** For each sample $i$ from $0$ to $N-1$:
    a. Calculate position $P_i = O + D \cdot t_i$.
    b. Calculate height $h_i = \|P_i\| - R_p$.
    c. Compute local extinction $\sigma_t(h_i)$ and scattering coefficients $\sigma_s^R(h_i), \sigma_s^M(h_i)$ [S6, unofficial].
    d. Compute optical depth from $P_i$ to the sun's outer boundary ($\tau_{sun}$) and from $P_i$ to the observer ($\tau_{view}$) [S1, S6, unofficial].
    e. Compute transmittances $Tr_{sun} = \exp(-\tau_{sun})$ and $Tr_{view} = \exp(-\tau_{view})$ [S1, unofficial].
    f. Compute source term contribution: $C_i = Tr_{view} \cdot Tr_{sun} \cdot (\sigma_s^R(h_i)P_R + \sigma_s^M(h_i)P_M) \cdot \Delta t$ [S6, unofficial].
    g. Accumulate scattered light: $L_{sky} += C_i \cdot L_{outerspace} \cdot \Omega_{sun}$ [S6, unofficial].
4.  **Direct Sun:** Add direct sun disk luminance if visible: $L_{sun} = disk(\theta) \cdot L_{outerspace} \cdot Tr(O \rightarrow outerspace)$ [S6, unofficial].
5.  **Composite:** Final color $C = L_{sky} + L_{sun}$ [S6, unofficial]. Apply tone mapping (e.g., ACES Filmic) [S1, S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_single_scattering_atmosphere(observer_pos, view_dir, sun_dir, params):
    """
    observer_pos: vec3, position of the camera/observer
    view_dir: vec3, normalized view direction
    sun_dir: vec3, normalized direction to the sun
    params: dict containing R_p, R_a, H_R, H_M, H_O, sigma_s_R0, sigma_s_M0, sigma_a_O0, L_outerspace, Omega_sun, N
    """
    R_p = params['R_p']
    R_a = params['R_a']
    H_R = params['H_R']
    H_M = params['H_M']
    H_O = params['H_O']
    sigma_s_R0 = params['sigma_s_R0']
    sigma_s_M0 = params['sigma_s_M0']
    sigma_a_O0 = params['sigma_a_O0']
    L_outerspace = params['L_outerspace']
    Omega_sun = params['Omega_sun']
    N = params['N']

    # 1. Ray-Atmosphere Intersection
    # Solve ||observer_pos + t * view_dir||^2 = R_a^2
    # a = 1, b = 2 * dot(observer_pos, view_dir), c = dot(observer_pos, observer_pos) - R_a^2
    a = 1.0
    b = 2.0 * dot(observer_pos, view_dir)
    c = dot(observer_pos, observer_pos) - R_a * R_a
    discriminant = b * b - 4 * a * c
    
    if discriminant < 0:
        return vec3(0.0) # No intersection

    sqrt_disc = sqrt(discriminant)
    t_in = (-b - sqrt_disc) / (2 * a)
    t_out = (-b + sqrt_disc) / (2 * a)

    # Handle observer inside planet or atmosphere
    if t_in < 0:
        t_in = 0.0 # Start from observer if inside atmosphere
    
    # If observer is inside planet, advance to surface exit (simplified)
    # Check intersection with planet surface
    c_planet = dot(observer_pos, observer_pos) - R_p * R_p
    disc_planet = b * b - 4 * a * c_planet
    if disc_planet >= 0:
        t_planet_in = (-b - sqrt(disc_planet)) / (2 * a)
        t_planet_out = (-b + sqrt(disc_planet)) / (2 * a)
        if t_planet_out > 0:
            # Observer inside planet, start at exit
            t_in = max(t_in, t_planet_out)
            t_out = max(t_out, t_planet_out)
    
    if t_in >= t_out:
        return vec3(0.0)

    ray_length = t_out - t_in
    if ray_length <= 0:
        return vec3(0.0)

    # 2. Integration
    scattered_light = vec3(0.0)
    
    # Precompute phase functions
    cos_theta = dot(view_dir, sun_dir)
    P_R = (3.0 / (16.0 * PI)) * (1.0 + cos_theta * cos_theta)
    g = 0.76
    denom = 1.0 + g * g - 2.0 * g * cos_theta
    P_M = (1.0 - g * g) / (4.0 * PI * pow(denom, 1.5))

    for i in range(N):
        # Sample position (uniform)
        t = t_in + (i + 0.5) * (ray_length / N)
        pos = observer_pos + view_dir * t
        height = length(pos) - R_p
        
        # Density and Coefficients
        rho_R = exp(-max(0.0, height) / H_R)
        rho_M = exp(-max(0.0, height) / H_M)
        rho_O = exp(-max(0.0, height) / H_O)
        
        sigma_s_R = sigma_s_R0 * rho_R
        sigma_s_M = sigma_s_M0 * rho_M
        sigma_a_O = sigma_a_O0 * rho_O
        
        sigma_t = sigma_s_R + 1.11 * sigma_s_M + sigma_a_O
        
        # Optical Depth to Sun
        # Ray from pos in sun_dir to atmosphere boundary
        b_sun = 2.0 * dot(pos, sun_dir)
        c_sun = dot(pos, pos) - R_a * R_a
        disc_sun = b_sun * b_sun - 4 * c_sun
        if disc_sun > 0:
            t_sun = (-b_sun - sqrt(disc_sun)) / 2.0
            if t_sun > 0:
                dist_to_sun = t_sun
            else:
                dist_to_sun = 0.0 # Sun direction doesn't exit atmosphere forward?
        else:
            dist_to_sun = 0.0
            
        # Integrate sigma_t from pos to sun_exit
        # Simplified: assume sigma_t is constant over the short segment or use midpoint
        # For accuracy, integrate along sun ray. Here we approximate with one step or loop.
        # Using one step for simplicity as per [S6, unofficial] snippet logic for j(x)
        sun_exit_pos = pos + sun_dir * dist_to_sun
        # Average sigma_t along sun path (midpoint)
        mid_sun = (pos + sun_exit_pos) / 2.0
        h_mid_sun = length(mid_sun) - R_p
        rho_R_mid = exp(-max(0.0, h_mid_sun) / H_R)
        rho_M_mid = exp(-max(0.0, h_mid_sun) / H_M)
        rho_O_mid = exp(-max(0.0, h_mid_sun) / H_O)
        sigma_t_mid = sigma_s_R0 * rho_R_mid + 1.11 * sigma_s_M0 * rho_M_mid + sigma_a_O0 * rho_O_mid
        
        tau_sun = sigma_t_mid * dist_to_sun
        Tr_sun = exp(-tau_sun)
        
        # Optical Depth to Observer (from pos to observer)
        # Distance is t (since observer is at 0 relative to ray start? No, observer is at origin of ray)
        # Distance from pos to observer is t
        tau_view = sigma_t * t # Approximation using local sigma_t
        Tr_view = exp(-tau_view)
        
        # Source Term Contribution
        # j(x) * Tr_view * dt
        # j(x) = L_outerspace * Tr_sun * Omega_sun * (sigma_s_R * P_R + sigma_s_M * P_M)
        # Note: Tr_sun in j(x) is from outer space to x. We approximated it as Tr(pos->sun_exit).
        # Actually, Tr(outerspace->x) = Tr(pos->sun_exit) if pos is on the ray from sun_exit to observer?
        # Standard formulation: Tr(outerspace->x) is transmittance from sun direction.
        
        scattering_coeff = sigma_s_R * P_R + sigma_s_M * P_M
        source = L_outerspace * Tr_sun * Omega_sun * scattering_coeff
        contribution = source * Tr_view * (ray_length / N)
        
        scattered_light += contribution

    # Direct Sun
    # Check if sun is visible
    angle = acos(clamp(cos_theta, -1.0, 1.0))
    sun_radius = radians(0.53 / 2.0)
    disk = 1.0 - smoothstep(sun_radius * 0.95, sun_radius * 1.05, angle)
    
    # Transmittance from observer to outer space in sun direction
    # Reuse dist_to_sun calculation logic
    b_sun_obs = 2.0 * dot(observer_pos, sun_dir)
    c_sun_obs = dot(observer_pos, observer_pos) - R_a * R_a
    disc_sun_obs = b_sun_obs * b_sun_obs - 4 * c_sun_obs
    if disc_sun_obs > 0:
        t_sun_obs = (-b_sun_obs - sqrt(disc_sun_obs)) / 2.0
        if t_sun_obs > 0:
            dist_obs_sun = t_sun_obs
        else:
            dist_obs_sun = 0.0
    else:
        dist_obs_sun = 0.0
        
    # Integrate sigma_t from observer to sun_exit
    sun_exit_obs = observer_pos + sun_dir * dist_obs_sun
    mid_obs = (observer_pos + sun_exit_obs) / 2.0
    h_mid_obs = length(mid_obs) - R_p
    rho_R_mid_obs = exp(-max(0.0, h_mid_obs) / H_R)
    rho_M_mid_obs = exp(-max(0.0, h_mid_obs) / H_M)
    rho_O_mid_obs = exp(-max(0.0, h_mid_obs) / H_O)
    sigma_t_mid_obs = sigma_s_R0 * rho_R_mid_obs + 1.11 * sigma_s_M0 * rho_M_mid_obs + sigma_a_O0 * rho_O_mid_obs
    tau_obs = sigma_t_mid_obs * dist_obs_sun
    Tr_obs = exp(-tau_obs)
    
    direct_sun = disk * L_outerspace * Tr_obs
    
    return scattered_light + direct_sun
```

## Subtopics

### [Sky View LUT with Angular Compression](6648-sky-view-lut-with-angular-compression.md)
A technique for rendering sky and atmosphere using a precomputed Look-Up Table (LUT) with angular compression, ported from a reference implementation. It builds on [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md) by storing precomputed radiance values in a compressed texture space for efficient runtime lookup.

### [Spectral Scattering Model for Atmosphere](9399-spectral-scattering-model-for-atmosphere.md)
This method visualizes the Earth from outer space by simulating light transport through a planetary atmosphere and large water bodies, accounting for Rayleigh and Mie scattering, direct illumination, indirect scattered light, interreflections between surface and volume, and seabed reflections. It sets itself apart from its siblings by employing fully spectral computations (precomputed into lookup tables) to achieve high realism in color reproduction, supporting arbitrarily dense atmospheres and water surfaces within a single precomputation framework for real-time GPU rendering. Unlike [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md), which focuses on single scattering approximations, this method computes multiple scattering orders (typically up to 6) and integrates water body optics.

### Water Scattering Precomputation
Uses different physical constants and parameterization for the water texture compared to atmospheric scattering [S3]. Incident light for water is attenuated through the atmosphere: $I_0^I(\lambda) = I^I(\lambda)e^{-t(P_g P, \lambda)}$ [S3]. The 4D water scattering texture is reduced to a 3D texture for rendering by using only the subset corresponding to observer depth $d=0$m [S3]. If observers below the water surface are needed, the full 4D texture can be used, fetching based on observer depth $d$ [S3]. For low accuracy requirements, only the first scattering order is needed for water because it is a weakly scattering medium [S3].

### Water Scattering Model
Water scattering model is similar to atmospheric but volume is between water level and seabed. Water has significant absorption (mainly red) and constant density [S3]. Terms for water are indicated with superscript 'water'. Density scale function $\rho$ is removed from equations [S3]. Water attenuation coefficient: $t^{\text{water}}(S, \lambda) = \|S\| (\zeta^a + \zeta^b)(\lambda)$ [S3]. Absorption and scattering coefficients for clear water ($\zeta_a, \zeta_s$) from [Buiteveld et al. 1994], sampled at 2nm [S3]. Mie scattering is not explicitly modeled in water because data for seawater is unavailable and it is strongly anisotropic (forward lobe), only noticeable looking towards light source. Since observers are above water level, it is neglected [S3].

### Jackèl and Walter Atmospheric Rendering
Highly realistic atmospheric rendering, allowing modeling of real world atmospheres (composed by ozone, haze, dust, soot, sulphur acid, etc.) [S8]. Combine four sub-models, consisting of a number of concentric shells, to determine atmospheric model: clear air, aerosol, ozone and rain layer [S8]. Account for both Rayleigh and Mie scattering [S8]. Approximate second order scattering to obtain more accurate results [S8]. Authors do not explain how the second order scattering approximation is performed [S8].

### Preetham Sky Model
Considered the rendering of outdoor scenes including sunlight and skylight, and the effects of aerial perspective (de-saturation and color shift of distant objects). Two inexpensive analytic models are developed [S8]. Compute the sky spectral radiance function for a set of sun positions and turbidities (using the method of Nishita et al. [59]), then fit a parametric function [S8]. Wilkie et al. propose a simple analytical model for skylight polarisation to be added to Preetham’s skylight model [S8].

### Jensen Nighttime Sky Rendering
Rendering of nighttime sky including illumination from the Moon, stars, zodiacal light, and atmosphere [S8]. Use the atmospheric model and scattering phase functions of Nishita [59] [S8]. Simulate transport with multiple scattering combining distribution ray-tracing and ray-marching [S8]. As the ray traverses the atmosphere, use ray-marching to integrate optical-depth sampling the in-scattered indirect radiance and the direct illumination in random positions [S8].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of integration steps (typically 32–64) [S6, unofficial].
*   **Memory Complexity:** $O(1)$ if computed per-pixel; $O(1)$ for LUT variants if texture size is fixed [S1, S6, unofficial].
*   **Performance:** Real-time on modern GPUs for $N \le 64$ [S6, unofficial].
*   **Strengths:** Physically plausible colors, efficient, handles sun position and atmospheric depth correctly [S6, unofficial].
*   **Limitations:** Ignores multiple scattering, leading to dark skies in deep atmospheres or near the horizon in high optical depth scenarios [S6, unofficial].
*   **When to Use:** Real-time sky rendering, games, VR [S6, unofficial].
*   **When Not to Use:** Cinematic rendering requiring high dynamic range and deep atmosphere realism, underwater scenes (use Water Scattering Model) [S3][S6, unofficial].

## Background
Originated from Nishita et al. (1993) "Display of the Earth from Space" [S3, S8]. It is a foundational method for realistic sky rendering in computer graphics. Used in real-time sky rendering, specifically referenced in Sébastien Hillaire's 2016 course and 2020 paper, and Scratch a Pixel tutorial [S6, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S2] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S3] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
[S4] Depersonalizc/volumetric-clouds, https://github.com/Depersonalizc/volumetric-clouds, fetched 2026-10-27 (unofficial)
[S5] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
[S6] Sky and Atmosphere rendering: A physical approach, https://cpp-rendering.io/sky-and-atmosphere-rendering/, fetched 2026-10-28 (unofficial)
[S7] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S8] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
