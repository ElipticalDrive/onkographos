# Cloud Volume Rendering

Also called: Basic Volumetric Rendering Clouds

## Overview
Cloud Volume Rendering is a direct volume ray casting technique that renders volumetric clouds by marching rays through a 3D density texture, applying detail noise for geometric carving, and computing multi-octave Henyey-Greenstein scattering [S1, unofficial]. It differs from standard front-to-back rendering by explicitly incorporating procedural noise-based detail carving and a specific multi-octave phase function approximation to simulate the complex light transport within cloud volumes [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: Ray origin $o$, ray direction $d$, camera parameters, time $t$, and three textures: `densityTexture` (highp sampler3D, RED/UNSIGNED_BYTE), `noiseTexture` (highp sampler3D, RED/UNSIGNED_BYTE), and `blueNoiseTexture` (sampler2D, RGBA/UNSIGNED_BYTE) [S1, unofficial].
*   **Outputs**: Final color $C_{out}$ and opacity $\alpha_{out}$ for each pixel.

**Data Structures**
*   **Textures**:
    *   `densityTexture`: 3D texture storing base cloud density. Format: `RED/UNSIGNED_BYTE` [S1, unofficial].
    *   `noiseTexture`: 3D texture storing detail noise for carving. Format: `RED/UNSIGNED_BYTE` [S1, unofficial].
    *   `blueNoiseTexture`: 2D texture for stochastic sampling or jittering. Format: `RGBA/UNSIGNED_BYTE` [S1, unofficial].
*   **Parameters**:
    *   `STEPS_PRIMARY`: 32 (main view ray steps) [S1, unofficial].
    *   `STEPS_LIGHT`: 16 (light ray steps) [S1, unofficial].
    *   `densityMultiplier`: Range $[0, \infty)$ [S1, unofficial].
    *   `detailSize`: Range $[0, \infty)$ [S1, unofficial].
    *   `detailStrength`: Range $[0, 1]$ [S1, unofficial].
    *   $\sigma_S$: Scattering coefficient [S1, unofficial].
    *   $\sigma_A$: Absorption coefficient [S1, unofficial].
    *   $\sigma_T = \sigma_S + \sigma_A$: Extinction coefficient [S1, unofficial].

**Formulas**
*   **Henyey-Greenstein Phase Function**:
    $$P(g, \theta) = \frac{1}{4\pi} \frac{1-g^2}{(1+g^2-2g\cos\theta)^{3/2}}$$
    where $g$ is the asymmetry parameter and $\theta$ is the scattering angle [S1, unofficial].

**Algorithm Steps**
1.  **Ray-AABB Intersection**:
    Calculate intersection with the cloud volume's Axis-Aligned Bounding Box (AABB) using the slab method:
    $$t_{min} = \frac{box_{min} - o}{d}, \quad t_{max} = \frac{box_{max} - o}{d}$$
    $$t_{near} = \max(t_{min}), \quad t_{far} = \min(t_{max})$$
    If the ray origin is inside the AABB, set $t_{near} = 10^{-4}$ [S1, unofficial].

2.  **Ray Marching Loop**:
    Iterate $i$ from $0$ to `STEPS_PRIMARY` - 1.
    *   Calculate step size $h = (t_{far} - t_{near}) / \text{STEPS\_PRIMARY}$.
    *   Current position $p = o + (t_{near} + i \cdot h) \cdot d$.
    *   **Detail Carving**:
        If `detailStrength` > 0:
        *   Sample detail noise: $\rho_{detail} = \text{texture}(\text{noiseTexture}, p / \text{detailSize})$.
        *   Animate position: $p_{anim} = p - \text{vec3}(0.1 \cdot t)$ [S1, unofficial].
        *   Sample base density: $\rho_{cloud} = \text{texture}(\text{densityTexture}, p_{anim})$.
        *   Remap density:
            $$\rho_{final} = \max(0, \min(1, \text{remap}(\rho_{cloud}, \text{detailStrength} \cdot \rho_{detail}, 1.0, 0.0, 1.0)))$$
            [S1, unofficial].
        *   Else: $\rho_{final} = \rho_{cloud}$.
    *   **Optical Depth**:
        Calculate optical depth for this step: $\tau = \sigma_T \cdot \rho_{final} \cdot h$.
    *   **Transmittance**:
        Update accumulated transmittance $T = T \cdot e^{-\tau}$.
    *   **Radiance Contribution**:
        Calculate light contribution using multi-octave Henyey-Greenstein scattering (see below).
        Add to accumulated color $C_{accum}$.
    *   **Alpha Accumulation**:
        Update accumulated alpha $\alpha_{accum} = \alpha_{accum} + T \cdot (1 - e^{-\tau})$.
    *   **Early Exit**:
        If $\alpha_{accum} > 1 - \epsilon$ (or $T < \epsilon$), break loop.

3.  **Multi-Octave Radiance Calculation**:
    For each step, compute radiance $L$ by looping 4 times (octaves).
    *   Initialize weights and coefficients.
    *   For octave $k \in \{0, 1, 2, 3\}$:
        *   Determine asymmetry parameter $g_k$. The sources specify mixing HG phase functions with $g \in \{-0.1c, 0.3c\}$ [S1, unofficial]. Note: The variable $c$ is not explicitly defined in the provided notes; it is assumed to be a constant or derived from context.
        *   Weight the contribution: The sources mention weighting by 0.7 and attenuating terms $a, b, c$ by factors 0.2, 0.5, 0.5 respectively [S1, unofficial].
        *   Calculate phase function $P(g_k, \theta)$ where $\theta$ is the angle between light direction and view direction.
        *   Accumulate radiance contribution based on $\sigma_S$, $P(g_k, \theta)$, and light transmittance (using `STEPS_LIGHT` steps for light ray).

4.  **Final Color**:
    $$C_{out} = C_{accum}$$
    $$\alpha_{out} = \alpha_{accum}$$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cloud(o, d, time, density_texture, noise_texture, blue_noise_texture,
                 STEPS_PRIMARY=32, STEPS_LIGHT=16,
                 density_multiplier=1.0, detail_size=1.0, detail_strength=0.5,
                 sigma_s=1.0, sigma_a=0.1):
    sigma_t = sigma_s + sigma_a
    
    # Ray-AABB Intersection
    t_min = (box_min - o) / d
    t_max = (box_max - o) / d
    t_near = max(t_min)
    t_far = min(t_max)
    
    if is_inside_box(o, box_min, box_max):
        t_near = 1e-4
        
    if t_near >= t_far:
        return (0.0, 0.0)
        
    step_size = (t_far - t_near) / STEPS_PRIMARY
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    transmittance = 1.0
    
    for i in range(STEPS_PRIMARY):
        if transmittance < 1e-4 or accumulated_alpha > 0.99:
            break
            
        t = t_near + i * step_size
        p = o + t * d
        
        # Base Density
        p_anim = p - vec3(0.1 * time)
        rho_cloud = texture(density_texture, p_anim / detail_size)
        
        # Detail Carving
        rho_final = rho_cloud
        if detail_strength > 0:
            rho_detail = texture(noise_texture, p_anim / detail_size)
            # Remap function: remap(value, min_val, max_val, out_min, out_max)
            # Note: The source formula uses detail_strength * rho_detail as a threshold/offset
            # Implementation depends on specific remap logic not fully detailed.
            # Assuming standard remap: (value - in_min) / (in_max - in_min) * (out_max - out_min) + out_min
            # Here, the source says: remap(rho_cloud, detail_strength * rho_detail, 1.0, 0.0, 1.0)
            # This implies mapping [detail_strength * rho_detail, 1.0] to [0.0, 1.0]
            rho_final = remap(rho_cloud, detail_strength * rho_detail, 1.0, 0.0, 1.0)
            rho_final = max(0.0, min(1.0, rho_final))
            
        rho_final *= density_multiplier
        
        # Optical Depth
        tau = sigma_t * rho_final * step_size
        
        # Transmittance
        T_step = exp(-tau)
        transmittance *= T_step
        
        # Alpha
        alpha_step = 1.0 - T_step
        accumulated_alpha += transmittance * alpha_step
        
        # Radiance Calculation (Multi-Octave HG)
        # Light ray marching for transmittance to sun
        light_dir = normalize(vec3(1.0, 1.0, 0.0)) # Example light direction
        light_transmittance = 1.0
        for j in range(STEPS_LIGHT):
            p_light = p + j * step_size * light_dir
            rho_light = texture(density_texture, p_light / detail_size)
            tau_light = sigma_t * rho_light * step_size
            light_transmittance *= exp(-tau_light)
            
        # Multi-Octave Phase Function
        radiance = vec3(0.0)
        # Loop 4 times as per source
        for k in range(4):
            # Determine g based on source: g in {-0.1c, 0.3c}
            # Assuming c is a constant, e.g., 1.0 or derived from octave
            c = 1.0 
            if k % 2 == 0:
                g = -0.1 * c
            else:
                g = 0.3 * c
                
            # Calculate scattering angle theta
            cos_theta = dot(normalize(d), normalize(light_dir))
            phase = heney_greenstein(g, cos_theta)
            
            # Weighting and attenuation
            # Source mentions weighting by 0.7 and attenuating terms a, b, c by 0.2, 0.5, 0.5
            # This part is ambiguous without explicit variable definitions for a, b, c.
            # Assuming a generic accumulation with decay
            weight = 0.7 * (0.5 ** k) 
            radiance += weight * phase * light_transmittance * sigma_s * rho_final
            
        accumulated_color += transmittance * radiance * alpha_step
        
    return (accumulated_color, accumulated_alpha)

def heney_greenstein(g, cos_theta):
    g_sq = g * g
    denom = 1.0 + g_sq - 2.0 * g * cos_theta
    return (1.0 / (4.0 * pi)) * (1.0 - g_sq) / (denom ** 1.5)

def remap(value, in_min, in_max, out_min, out_max):
    if in_max == in_min:
        return out_min
    t = (value - in_min) / (in_max - in_min)
    return out_min + t * (out_max - out_min)
```

**Edge Cases and Pitfalls**
*   **Precision**: Using `UNSIGNED_BYTE` textures limits density resolution to 256 levels. High `densityMultiplier` may cause banding [S1, unofficial].
*   **Boundaries**: Ray-AABB intersection must handle rays originating inside the volume correctly by setting $t_{near} = 10^{-4}$ [S1, unofficial].
*   **Degenerate Input**: If `detailStrength` is 0, detail noise sampling is skipped. If `detailSize` is 0, division by zero occurs in texture coordinate calculation [S1, unofficial].

## Subtopics

### Front-to-Back Volume Rendering
The foundational blending approach used here. It accumulates color and alpha from near to far. Choose when transparency is needed and order-independent transparency is not critical. [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md)

### Ray Marching Volume Rendering with Henyey-Greenstein Scattering
Provides the specific phase function implementation. Choose when accurate single-scattering approximation is required. [Ray Marching Volume Rendering with Henyey-Greenstein Scattering](2545-ray-marching-volume-rendering-with-henyey-greenstein-scatter.md)

### Fixed Step Ray Marching
The discretization method used for the ray traversal. Choose when uniform sampling is sufficient and adaptive stepping is too complex. [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md)

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$ where $N$ is `STEPS_PRIMARY` and $M$ is `STEPS_LIGHT` per pixel. With $N=32$ and $M=16$, this is $512$ texture lookups and phase function evaluations per pixel [S1, unofficial].
*   **Memory Complexity**: $O(V)$ for the 3D textures, where $V$ is the volume resolution. The textures are `UNSIGNED_BYTE`, so memory usage is low per voxel [S1, unofficial].
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Simple implementation, uses standard 3D textures, provides visually plausible cloud details via noise carving [S1, unofficial].
*   **Limitations**: Single-scattering approximation may look dark in dense clouds. Fixed step size may lead to aliasing or slow convergence in sparse regions. The multi-octave HG approximation is heuristic and may not match physical accuracy [S1, unofficial].
*   **When to Use**: Real-time rendering of volumetric clouds where performance is critical and high-fidelity multi-scattering is not required.
*   **When Not to Use**: Offline rendering requiring physical accuracy, or scenes with highly complex light transport (e.g., deep shadows, multiple scattering).

## Background
Cloud Volume Rendering, as described in the source, is an implementation detail from the `al-ro/volumetrics` project [S1, unofficial]. It builds upon the principles of direct volume ray casting and Henyey-Greenstein scattering, adapting them for real-time cloud rendering using WebGL. The specific multi-octave phase function mixing and detail carving technique are characteristic of this implementation [S1, unofficial].

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
