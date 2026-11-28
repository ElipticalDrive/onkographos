# Ray Marching with Single Scattering

Also called: Real-time Volumetric Cloud Rendering, Real-time Volumetric Rendering, Volumetric Cloud Rendering, Volumetric Cloud Rendering with Ray Marching

## Overview
Ray marching with single scattering is a real-time technique for rendering volumetric clouds by discretizing the Radiative Transfer Equation (RTE) along view rays and approximating light transport with a single scattering event [S3, unofficial]. It combines procedural noise for density generation with a physically-based lighting model to produce dynamic, three-dimensional cloudscapes [S1, S3, unofficial]. It is distinguished from multi-scattering methods by its computational efficiency, trading physical accuracy for performance, and from billboard-based methods by its ability to render volumetric depth and internal structure [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera origin $\mathbf{o}$, normalized view direction $\mathbf{d}$, sun direction $\mathbf{d}_{\text{sun}}$, sun color $\mathbf{C}_{\text{sun}}$, ambient color $\mathbf{C}_{\text{amb}}$, cloud bounding box (AABB), cloud layer height limits ($h_{\min}, h_{\max}$), and procedural noise textures (Weather Map, Base Noise, Edge Detail) [S1, S3, unofficial].
*   **Outputs:** Final pixel color $C_{\text{final}}$ [S1, unofficial].

**Data Structures:**
1.  **Weather Map (2D Texture):**
    *   Format: RGBA, Float32 [S2, unofficial].
    *   Red Channel: Base cloud coverage ($0.0$ to $1.0$) [S3, unofficial].
    *   Blue Channel: Cloud type ($0.0$=stratus, $0.5$=stratocumulus, $1.0$=cumulus) [S3, unofficial].
2.  **Base Noise Texture (3D Texture, $128 \times 128 \times 128$):**
    *   Red Channel: Combined Perlin-Worley noise [S3, unofficial].
    *   Green, Blue, Alpha Channels: Low-frequency inverse Worley noise at increasing frequencies [S3, unofficial].
3.  **Edge Detail Texture (3D Texture, $32 \times 32 \times 32$):**
    *   Channels: High-frequency inverse Worley noise at increasing frequencies [S3, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Compute ray direction $\mathbf{d} = \text{normalize}(\mathbf{s} - \mathbf{o})$, where $\mathbf{s}$ is the screen position and $\mathbf{o}$ is the eye position [S1, unofficial].
    *   Compute sky color using $\mathbf{d}$ [S1, unofficial].
    *   Initialize main transmittance $T = 1.0$, accumulated color $C = 0$, and distance $t = 0.0$ [S1, unofficial].
    *   Set scene variables (scattering/absorption coefficients) based on time of day [S1, unofficial].
2.  **Main Ray Marching Loop:**
    *   Iterate $i$ from $0$ to `MAX_STEPS` [S1, unofficial].
    *   Calculate position $\mathbf{p} = \mathbf{o} + t \mathbf{d}$ [S1, unofficial].
    *   **Bounding Check:** If $\mathbf{p}$ is outside the cloud bounding box or cloud layer height limits, skip or break [S3, unofficial].
    *   **Density Evaluation:**
        *   Sample base noise $\rho_{\text{base}}$ from the 3D texture at $\mathbf{p}$ [S3, unofficial].
        *   Sample edge detail $\rho_{\text{edge}}$ from the 32x32x32 texture at $\mathbf{p}$ [S3, unofficial].
        *   Combine noise: Use fBm of low-frequency inverse Worley channels to modulate the Perlin-Worley noise [S3, unofficial].
        *   Apply height modulation: Modulate density with a linear gradient from $h_{\min}$ to $h_{\max}$ [S3, unofficial].
        *   Apply weather map: Scale density by coverage from the Weather Map Red channel [S3, unofficial].
        *   Final density $\rho = \text{clamp}(\text{combined noise}, 0.0, 1.0)$ [S1, unofficial].
    *   **Optimization:** If $\rho < 0.01$, increment $t$ by `STEP_SIZE` and continue [S1, unofficial].
    *   **Light Marching (Single Scattering):**
        *   Initialize light transmittance $T_{\text{light}} = 1.0$, light distance $t_{\text{light}} = 0.0$ [S1, unofficial].
        *   Compute light direction $\mathbf{d}_{\text{light}} = \text{normalize}(\mathbf{d}_{\text{sun}} - \mathbf{p})$ [S1, unofficial].
        *   Iterate $j$ from $0$ to `MAX_STEPS_LIGHT` [S1, unofficial].
        *   Calculate light position $\mathbf{p}_{\text{light}} = \mathbf{p} + t_{\text{light}} \mathbf{d}_{\text{light}}$ [S1, unofficial].
        *   Sample density $\rho_{\text{light}}$ at $\mathbf{p}_{\text{light}}$ [S1, unofficial].
        *   Update light transmittance: $T_{\text{light}} \leftarrow T_{\text{light}} \cdot e^{-\rho_{\text{light}} \cdot \text{extinctionCoeff} \cdot \text{STEP\_SIZE\_LIGHT} \cdot 1.5}$ [S1, unofficial].
        *   If $T_{\text{light}} < 0.05$, break light loop [S1, unofficial].
        *   Increment $t_{\text{light}}$ by `STEP_SIZE_LIGHT` [S1, unofficial].
    *   **Scattering Calculation:**
        *   Compute phase function value $f_p$ using $\mathbf{d}$ and $\mathbf{d}_{\text{light}}$ [S1, unofficial].
        *   Compute $L_s = \mathbf{C}_{\text{sun}} \cdot T_{\text{light}} \cdot f_p \cdot 6.0$ [S1, unofficial].
        *   Compute $L_a = \mathbf{C}_{\text{amb}} \cdot 0.7$ [S1, unofficial].
        *   Compute local scattering contribution: $S = (L_s + L_a) \cdot T \cdot \rho \cdot \text{STEP\_SIZE}$ [S1, unofficial].
    *   **Accumulation:**
        *   $C \leftarrow C + S$ [S1, unofficial].
        *   Update main transmittance: $T \leftarrow T \cdot e^{-\rho \cdot \text{extinctionCoeff} \cdot \text{STEP\_SIZE} \cdot 1.5}$ [S1, unofficial].
        *   If $T < 0.01$, break main loop [S1, unofficial].
        *   Increment $t$ by `STEP_SIZE` [S1, unofficial].
        *   If $t > 20.0$, break main loop [S1, unofficial].
3.  **Final Composition:**
    *   $C_{\text{final}} = C + T \cdot \text{sky}$ [S1, unofficial].
    *   Output $C_{\text{final}}$ [S1, unofficial].

**Formulas:**
*   **Henyey-Greenstein Phase Function:**
    $$ f_p(\cos \theta) = \left( \text{mix}(F, B, 0.4) \right) \cdot 0.4 $$
    Where:
    $$ F = \frac{1 - g_1^2}{(1 + g_1^2 - 2 g_1 \cos \theta)^{1.5}} $$
    $$ B = \frac{1 - g_2^2}{(1 + g_2^2 - 2 g_2 \cos \theta)^{1.5}} $$
    [S1, unofficial]
*   **In-scattered Radiance ($L_s$):**
    $$ L_s = L_{\text{sun}} T f_p $$
    [S3, unofficial]
*   **Ambient Radiance ($L_a$):**
    $$ L_a = L_{\text{ambient}} (1 - e^{-\rho}) $$
    [S3, unofficial]
*   **Transmittance Update:**
    $$ T_k = T_{k-1} e^{-\alpha \rho \Delta s} $$
    [S3, unofficial]
*   **Radiance Update:**
    $$ L_k = L_{k-1} + (L_a + L_s) T_k \Delta s $$
    [S3, unofficial]
*   **Sky Color:**
    $$ \text{sky} = \text{mix}(\text{skyBottom}, \text{skyTop}, \text{clamp}(\mathbf{d}_y \cdot 0.5 + 0.5, 0.0, 1.0)) $$
    [S1, unofficial]

**Parameters:**
*   `MAX_STEPS`: 50 [S1, unofficial].
*   `STEP_SIZE`: 0.08 [S1, unofficial].
*   `MAX_STEPS_LIGHT`: 5 [S1, unofficial].
*   `STEP_SIZE_LIGHT`: 0.6 [S1, unofficial].
*   `scatteringCoeff`: 1.5 (sunset), 1.2 (day) [S1, unofficial].
*   `absorptionCoeff`: 0.4 (sunset), 0.2 (day) [S1, unofficial].
*   `extinctionCoeff`: `scatteringCoeff` + `absorptionCoeff` [S1, unofficial].
*   `alpha` ($\alpha$): Light absorption coefficient [S3, unofficial].
*   `g1`: 0.7 (forward anisotropy) [S1, unofficial].
*   `g2`: -0.2 (backward anisotropy) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision & Boundaries:** The algorithm uses a fixed maximum distance of $20.0$ and a transmittance threshold of $0.01$ to terminate early [S1, unofficial]. WebGL 1 does not support dynamic loop bounds, requiring `STEP_SIZE` and `STEP_SIZE_LIGHT` to be hardcoded constants [S2, unofficial].
*   **Degenerate Input:** Aliasing artifacts become noticeable when the camera is positioned inside the cloud volume [S3, unofficial]. The method cannot render two clouds vertically layered at the same position [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_single_scattering(camera_origin, camera_dir, sun_dir, sun_color, ambient_color, 
                                weather_map, base_noise, edge_detail, bounds_min, bounds_max,
                                h_min, h_max, max_steps=50, step_size=0.08, max_steps_light=5, step_size_light=0.6):
    # Initialize
    transmittance = 1.0
    accumulated_color = vec3(0.0)
    t = 0.0
    
    # Sky color calculation
    sky_top = vec3(0.2, 0.4, 0.8) # Day default
    sky_bottom = vec3(0.7, 0.85, 1.0) # Day default
    screen_range = clamp(camera_dir.y * 0.5 + 0.5, 0.0, 1.0)
    sky = mix(sky_bottom, sky_top, screen_range)
    
    # Scene variables (simplified for day)
    scattering_coeff = 1.2
    absorption_coeff = 0.2
    extinction_coeff = scattering_coeff + absorption_coeff
    
    for i in range(max_steps):
        p = camera_origin + t * camera_dir
        
        # Bounding box check
        if not is_inside_box(p, bounds_min, bounds_max):
            t += step_size
            continue
            
        # Height check
        if p.y < h_min or p.y > h_max:
            t += step_size
            continue
            
        # Density evaluation
        # 1. Base noise (Perlin-Worley)
        density = sample_base_noise(p)
        
        # 2. Edge detail erosion
        edge = sample_edge_detail(p)
        density *= edge # Simplified erosion
        
        # 3. Weather map modulation
        weather = sample_weather_map(p.xy) # Project to 2D
        density *= weather.r # Coverage
        
        # 4. Height modulation
        height_factor = smoothstep(h_min, h_max, p.y)
        density *= height_factor
        
        density = clamp(density, 0.0, 1.0)
        
        # Optimization: Skip low density
        if density < 0.01:
            t += step_size
            continue
            
        # Light marching
        light_dir = normalize(sun_dir - p)
        transmittance_light = 1.0
        t_light = 0.0
        
        for j in range(max_steps_light):
            p_light = p + t_light * light_dir
            density_light = sample_base_noise(p_light) # Simplified: using same noise for light path
            
            # Beer-Lambert for light
            transmittance_light *= exp(-density_light * extinction_coeff * step_size_light * 1.5)
            
            if transmittance_light < 0.05:
                break
                
            t_light += step_size_light
            
        # Scattering
        cos_theta = dot(camera_dir, light_dir)
        phase = henyey_greenstein(cos_theta, g1=0.7, g2=-0.2)
        
        # In-scattered light
        ls = sun_color * transmittance_light * phase * 6.0
        la = ambient_color * 0.7
        
        # Accumulate
        scattered = ls + la
        accumulated_color += transmittance * density * scattered * step_size
        
        # Update transmittance
        transmittance *= exp(-density * extinction_coeff * step_size * 1.5)
        
        if transmittance < 0.01:
            break
            
        t += step_size
        
        if t > 20.0:
            break
            
    final_color = accumulated_color + transmittance * sky
    return final_color

def henyey_greenstein(cos_theta, g1=0.7, g2=-0.2):
    denom1 = 1.0 + g1*g1 - 2.0*g1*cos_theta
    forward = (1.0 - g1*g1) / pow(denom1, 1.5)
    
    denom2 = 1.0 + g2*g2 - 2.0*g2*cos_theta
    backward = (1.0 - g2*g2) / pow(denom2, 1.5)
    
    return mix(forward, backward, 0.4) * 0.4

def is_inside_box(p, min, max):
    return p.x >= min.x and p.x <= max.x and p.y >= min.y and p.y <= max.y and p.z >= min.z and p.z <= max.z

def sample_base_noise(p):
    # Returns density from 3D texture (128x128x128)
    return texture3D(base_noise, p).r

def sample_edge_detail(p):
    # Returns erosion factor from 3D texture (32x32x32)
    return texture3D(edge_detail, p).r

def sample_weather_map(p):
    # Returns coverage from 2D texture
    return texture2D(weather_map, p)
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ per pixel, where $N$ is the number of main ray steps and $M$ is the number of light ray steps [S1, unofficial].
*   **Memory Complexity:** Dominated by the storage of the 3D noise textures ($128^3$ and $32^3$) and the 2D weather map [S3, unofficial].
*   **Performance:** Achieves 2.6 ms frame time at 1440p on an AMD Ryzen 5 7600X CPU and AMD Radeon RX 7900 XT GPU [S3, unofficial].
*   **Strengths:** Real-time performance; dynamic wind simulation via texture scrolling; configurable cloud types and weather patterns [S3, unofficial].
*   **Limitations:** Single scattering approximation lacks physical accuracy for thick clouds [S1, S3, unofficial]. Noticeable aliasing occurs when the camera is inside the cloud volume [S3, unofficial]. The approach cannot render two clouds vertically layered at the same position [S3, unofficial].
*   **Scaling:** Performance scales with cloud layer thickness; reducing thickness yields major performance gains by enabling smaller/fewer ray marching steps [S3, unofficial].
*   **When to use:** Suitable for real-time applications requiring dynamic, volumetric cloudscapes where performance is prioritized over strict physical accuracy [S3, unofficial].
*   **When not to use:** Unsuitable for offline rendering requiring multiple scattering, or for close-up camera shots inside the volume where aliasing becomes prominent [S3, unofficial].

## Background
The method originates from real-time volumetric rendering techniques for games and simulations, specifically focusing on cloud rendering [S3, unofficial]. It builds on the principles of ray marching and procedural noise [S1, unofficial]. The specific implementation details, such as the combination of Perlin-Worley noise and the Henyey-Greenstein phase function, are documented in recent academic and open-source projects [S1, S3, unofficial].

## Sources
[S1] gmanishaa/volumetric-rendering, https://github.com/gmanishaa/volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
[S3] Real-time volumetric cloud rendering for games and simulations, https://erk.fe.uni-lj.si/2025/papers/loboda(real_time_volumetric).pdf, fetched 2026-10-29 (unofficial)
