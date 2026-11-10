# Ray Marching with Single Scattering

Also called: Real-time Volumetric Cloud Rendering, Real-time Volumetric Rendering, Volumetric Cloud Rendering, Volumetric Cloud Rendering with Ray Marching

## Overview
Ray marching with single scattering is a real-time technique for rendering volumetric clouds by discretizing the Radiative Transfer Equation along view rays [S1, unofficial][S3, unofficial]. It combines procedural noise for density generation with a physically-based single-scattering illumination model to produce dynamic, three-dimensional cloudscapes [S3, unofficial]. It is distinguished from [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) by its use of explicit ray marching through a sampled density field rather than implicit SDF evaluation, and from [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md) by its focus on discrete cloud layers with specific weather-map control rather than global atmospheric scattering [S2, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position $\mathbf{o}$ and direction $\mathbf{d}$; Sun position $\mathbf{p}_{sun}$; Cloud bounding box (min/max heights, horizontal bounds); Procedural noise parameters (wind, scale, density); Lighting parameters (sun color, ambient color, scattering/absorption coefficients).
*   **Outputs:** Final pixel color $C_{final}$.

**Data Structures:**
*   **Weather Map (2D Texture):** Stores large-scale cloud control.
    *   Red channel: Base cloud coverage ($0.0$ to $1.0$).
    *   Blue channel: Cloud type ($0.0$ stratus, $0.5$ stratocumulus, $1.0$ cumulus) [S3, unofficial].
*   **Base Noise Texture (3D, $128 \times 128 \times 128$):**
    *   Red channel: Combined Perlin-Worley noise ($\text{Perlin} \times \text{InverseWorley}$) [S3, unofficial].
    *   Channels G, B, A: Low-frequency inverse Worley noises at increasing frequencies [S3, unofficial].
*   **Edge Detail Texture (3D, $32 \times 32 \times 32$):**
    *   Channels R, G, B: High-frequency inverse Worley noises at increasing frequencies for edge erosion [S3, unofficial].
*   **Floating Point Texture (Optional):** Used in some implementations for density maps, stored as `Float32Array` with RGBA order, index $(x + y \cdot \text{width}) \cdot 4$ [S2, unofficial].

**Parameters:**
*   `MAX_STEPS`: Maximum main ray steps ($50$) [S1, unofficial].
*   `STEP_SIZE`: Main ray step size ($0.08$) [S1, unofficial].
*   `MAX_STEPS_LIGHT`: Maximum light ray steps ($5$) [S1, unofficial].
*   `STEP_SIZE_LIGHT`: Light ray step size ($0.6$) [S1, unofficial].
*   `scatteringCoeff`: $1.5$ (sunset), $1.2$ (day) [S1, unofficial].
*   `absorptionCoeff`: $0.4$ (sunset), $0.2$ (day) [S1, unofficial].
*   `extinctionCoeff`: `scatteringCoeff + absorptionCoeff` [S1, unofficial].
*   `cloudLayerThickness`: Vertical extent of the cloud volume (e.g., $15$ km) [S3, unofficial].

**Algorithm Steps:**

1.  **Ray Setup:** For each pixel, compute the view ray direction $\mathbf{d} = \text{normalize}(\mathbf{s} - \mathbf{o})$, where $\mathbf{s}$ is the screen position and $\mathbf{o}$ is the camera origin [S1, unofficial]. Compute the background sky color $C_{sky}$ by blending `skyTop` and `skyBottom` based on $\mathbf{d}.y$ [S1, unofficial].
2.  **Initialize Accumulators:** Set main transmittance $T = 1.0$, accumulated color $C_{accum} = \mathbf{0}$, and distance $t = 0.0$ [S1, unofficial].
3.  **Main Ray Marching Loop:** Iterate $i$ from $0$ to `MAX_STEPS`:
    a.  **Position:** Compute sample point $\mathbf{p} = \mathbf{o} + t \cdot \mathbf{d}$ [S1, unofficial].
    b.  **Boundary Check:** If $\mathbf{p}$ is outside the cloud bounding box (e.g., outside a sphere defined by SDF or above/below cloud layer heights), skip to step 3g [S1, unofficial][S3, unofficial].
    c.  **Density Evaluation:**
        i.  Evaluate base density $\rho_{base}$ using the 3D noise textures. Combine low-frequency inverse Worley channels (G, B, A) via weighted dot product to form an fBm value. Use this fBm to remap the Perlin-Worley noise in the Red channel [S3, unofficial].
        ii. Apply height modulation: Increase density from bottom to top of the cloud layer [S3, unofficial].
        iii. Apply weather map influence: Scale density by coverage (Red channel) and adjust shape by type (Blue channel) [S3, unofficial].
        iv. Evaluate edge detail: Sample the $32 \times 32 \times 32$ texture and use it to erode cloud edges, softening the appearance [S3, unofficial].
        v.  Final density $\rho = \text{clamp}(\rho_{base}, 0.0, 1.0)$ [S1, unofficial].
    d.  **Low Density Optimization:** If $\rho < 0.01$, increment $t$ by `STEP_SIZE` and continue to next iteration [S1, unofficial].
    e.  **Light Marching (Single Scattering):**
        i.  Initialize light transmittance $T_L = 1.0$ and light distance $t_L = 0.0$ [S1, unofficial].
        ii. Compute light direction $\mathbf{d}_L = \text{normalize}(\mathbf{p}_{sun} - \mathbf{p})$ [S2, unofficial].
        iii. Iterate $j$ from $0$ to `MAX_STEPS_LIGHT`:
            1.  Compute light sample point $\mathbf{p}_L = \mathbf{p} + t_L \cdot \mathbf{d}_L$ [S1, unofficial].
            2.  Evaluate density $\rho_L$ at $\mathbf{p}_L$ using the same noise evaluation as step 3c [S1, unofficial].
            3.  Update light transmittance: $T_L \leftarrow T_L \cdot e^{-\rho_L \cdot \text{extinctionCoeff} \cdot \text{STEP\_SIZE\_LIGHT} \cdot 1.5}$ [S1, unofficial].
            4.  If $T_L < 0.05$, break light loop [S1, unofficial].
            5.  If $\mathbf{p}_L$ exits the bounding box, break light loop [S2, unofficial].
            6.  Increment $t_L$ by `STEP_SIZE_LIGHT` [S1, unofficial].
    f.  **Scattering Calculation:**
        i.  Compute phase function $P(\theta)$ using Henyey-Greenstein with anisotropy $g=0.7$ (forward) and $g=-0.2$ (backward), mixed with weight $0.4$ [S1, unofficial].
        ii. Compute in-scattered radiance $L_s = C_{sun} \cdot T_L \cdot P(\theta)$ [S3, unofficial].
        iii. Compute ambient radiance $L_a = C_{ambient} \cdot (1 - e^{-\rho})$ [S3, unofficial].
        iv. Compute local scattering contribution: $C_{scatter} = (L_s + L_a) \cdot T \cdot \text{STEP\_SIZE}$ [S3, unofficial]. Note: Some implementations use $C_{scatter} = (1 - e^{-\rho \cdot \text{extinctionCoeff} \cdot \text{STEP\_SIZE}}) \cdot T_L \cdot C_{sun}$ [S2, unofficial].
    g.  **Accumulation:**
        i.  Update accumulated color: $C_{accum} \leftarrow C_{accum} + C_{scatter}$ [S1, unofficial].
        ii. Update main transmittance: $T \leftarrow T \cdot e^{-\rho \cdot \text{extinctionCoeff} \cdot \text{STEP\_SIZE} \cdot 1.5}$ [S1, unofficial].
    h.  **Early Exit:** If $T < 0.01$ or $t > 20.0$, break main loop [S1, unofficial].
    i.  **Step Increment:** Increment $t$ by `STEP_SIZE` [S1, unofficial].
4.  **Final Composition:** $C_{final} = C_{accum} + C_{sky} \cdot T$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_single_scattering(camera_origin, camera_dir, sun_pos, sun_color, ambient_color, cloud_bounds, noise_textures, weather_map, time):
    # Constants
    MAX_STEPS = 50
    STEP_SIZE = 0.08
    MAX_STEPS_LIGHT = 5
    STEP_SIZE_LIGHT = 0.6
    SCATTERING_COEFF = 1.5
    ABSORPTION_COEFF = 0.4
    EXTINCTION_COEFF = SCATTERING_COEFF + ABSORPTION_COEFF
    
    # Initialize
    transmittance = 1.0
    accumulated_color = vec3(0.0)
    t = 0.0
    
    # Sky color
    sky_color = calculate_sky_color(camera_dir)
    
    for i in range(MAX_STEPS):
        p = camera_origin + t * camera_dir
        
        # Check bounds (simplified sphere check as per S1)
        dist = sdf_sphere(p, 2.0)
        if dist <= 0.0:
            t += STEP_SIZE
            continue
            
        # Density evaluation
        density = evaluate_density(p, noise_textures, weather_map, time, cloud_bounds)
        
        # Low density optimization
        if density < 0.01:
            t += STEP_SIZE
            continue
            
        # Light marching
        transmittance_light = 1.0
        t_light = 0.0
        sun_dir = normalize(sun_pos - p)
        
        for j in range(MAX_STEPS_LIGHT):
            p_light = p + t_light * sun_dir
            density_light = evaluate_density(p_light, noise_textures, weather_map, time, cloud_bounds)
            
            # Update light transmittance
            transmittance_light *= exp(-density_light * EXTINCTION_COEFF * STEP_SIZE_LIGHT * 1.5)
            
            if transmittance_light < 0.05:
                break
                
            t_light += STEP_SIZE_LIGHT
            
        # Phase function
        cos_theta = dot(camera_dir, sun_dir)
        phase = henyey_greenstein(cos_theta)
        
        # Scattering
        # Using S3 formula for radiance update
        in_scattered = sun_color * transmittance_light * phase
        ambient = ambient_color * (1.0 - exp(-density))
        scattering = (in_scattered + ambient) * transmittance * STEP_SIZE
        
        accumulated_color += scattering
        
        # Update main transmittance
        transmittance *= exp(-density * EXTINCTION_COEFF * STEP_SIZE * 1.5)
        
        if transmittance < 0.01 or t > 20.0:
            break
            
        t += STEP_SIZE
        
    final_color = accumulated_color + sky_color * transmittance
    return final_color

def evaluate_density(p, textures, weather, time, bounds):
    # Height modulation
    height_factor = map(p.y, bounds.min_y, bounds.max_y, 0.0, 1.0)
    
    # Base noise from 128^3 texture
    # Red: Perlin * InverseWorley
    # G,B,A: Low-freq InverseWorley
    fbm = weighted_dot_product(textures.base_noise[p].g, textures.base_noise[p].b, textures.base_noise[p].a)
    base_noise = textures.base_noise[p].r * fbm
    
    # Weather map influence
    coverage = weather_map[uv_from_xy(p)].r
    cloud_type = weather_map[uv_from_xy(p)].b
    
    # Edge detail from 32^3 texture
    edge_detail = textures.edge_noise[p]
    
    density = base_noise * coverage * height_factor * edge_detail
    return clamp(density, 0.0, 1.0)

def henyey_greenstein(cos_theta):
    g1 = 0.7
    g2 = -0.2
    denom1 = 1.0 + g1*g1 - 2.0*g1*cos_theta
    forward = (1.0 - g1*g1) / pow(denom1, 1.5)
    denom2 = 1.0 + g2*g2 - 2.0*g2*cos_theta
    backward = (1.0 - g2*g2) / pow(denom2, 1.5)
    return mix(forward, backward, 0.4) * 0.4

def sdf_sphere(p, r):
    return -length(p) + r
```

## Subtopics

### SDF Cloud Rendering
Uses Signed Distance Functions to define cloud shapes implicitly, avoiding explicit ray marching through density fields. Choose when memory is constrained and simple shapes suffice. [S2, unofficial]

### Volume Ray Marching with Temporal Accumulation
Adds temporal stability to reduce flickering in noisy noise-based volumes. Choose when animation stability is critical. [S2, unofficial]

### Atmospheric Ray Marching
Focuses on global atmospheric scattering (Rayleigh/Mie) rather than discrete cloud layers. Choose for skybox rendering rather than volumetric clouds. [S2, unofficial]

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ per pixel, where $N$ is the number of main ray steps and $M$ is the number of light ray steps [S1, unofficial].
*   **Memory Complexity:** Dominated by the storage of 3D noise textures ($128^3$ and $32^3$) and the weather map [S3, unofficial].
*   **Performance:** Achieves $2.6$ ms frame time at $1440p$ on an AMD Ryzen 5 7600X / RX 7900 XT [S3, unofficial].
*   **Strengths:** Real-time performance; physically-based lighting; dynamic wind simulation; configurable cloud types [S3, unofficial].
*   **Limitations:** Single scattering approximation lacks multiple scattering effects, reducing physical accuracy [S2, unofficial]. Aliasing occurs when the camera is inside the volume [S3, unofficial]. Cannot render vertically layered clouds at the same position [S3, unofficial]. WebGL 1 requires hardcoded step sizes [S2, unofficial].

## Background
The technique originates from real-time volumetric rendering research for games and simulations, combining procedural noise with single-scattering models [S3, unofficial]. It builds on the foundational work of ray marching for volumetric data [S1, unofficial]. Key implementations include the OpenGL-based demo by gmanishaa [S1, unofficial] and the WebGPU-based system by Loboda et al. [S3, unofficial].

## Sources
[S1] gmanishaa/volumetric-rendering, https://github.com/gmanishaa/volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
[S3] Real-time volumetric cloud rendering for games and simulations, https://erk.fe.uni-lj.si/2025/papers/loboda(real_time_volumetric).pdf, fetched 2026-10-29 (unofficial)
