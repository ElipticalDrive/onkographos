# Emission-Absorption Ray Marching

Also called: Absorption plus Emission Volume Rendering, Direct Volume Rendering (Emission plus Absorption), EAM Render, EmissionAbsorption, X-Ray Rendering

## Overview
Emission-Absorption Ray Marching is a deterministic volume rendering technique that computes the final pixel color by integrating the combined effects of light attenuation (absorption/extinction) and light generation (emission) along a viewing ray [S1, unofficial][S6, S7]. It solves the problem of rendering participating media where particles both occlude background light and contribute their own luminance, such as clouds, smoke, or medical tissue [S6]. Unlike the [Absorption-only model](839-absorption-only-model.md), which only darkens the background, or the [Emission-only Model](1016-emission-only-model.md), which lacks intensity bounds, this method provides a physically bounded result by balancing source terms against exponential decay [S6]. It builds on the general [Optical models](33-optical-models.md) framework by solving the specific differential equation for media with non-zero extinction and emission coefficients.

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\boldsymbol{\omega}$ (unit vector).
*   Volume bounds (entry and exit distances $t_{min}, t_{max}$ or a bounding box).
*   Step size $h$ (or step count limit).
*   Extinction coefficient $\sigma_t$ (or opacity $\alpha$ per sample).
*   Emission coefficient $g(s)$ (or source term).
*   Background radiance $L_{bg}$ (or initial intensity $I_0$).

**Outputs:**
*   Final accumulated radiance $L_{out}$ and opacity $\alpha_{out}$.

**Data Structures:**
*   Accumulator: A vector or struct holding current accumulated color $C_{acc}$ and opacity $\alpha_{acc}$. Initialized to $(0, 0)$.
*   Transmission buffer (optional): An array storing cumulative transmission $T_i = \prod_{k=0}^{i-1} (1-\alpha_k)$ for back-to-front compositing [S3, unofficial][S7].

**Numbered Steps:**

1.  **Initialization:**
    *   Calculate ray length: $L = t_{max} - t_{min}$.
    *   Determine step size: $h = L / N$ or use a fixed $h$ [S2, S4, unofficial].
    *   Initialize random offset for jittering (optional): $t_{offset} = \text{rand}() \cdot h$ [S2, unofficial].
    *   Initialize accumulator: $C_{acc} = \mathbf{0}$, $\alpha_{acc} = 0$.
    *   Initialize transmission: $T_{curr} = 1.0$ [S7].

2.  **Ray Marching Loop:**
    *   Iterate $i$ from $0$ to $N-1$:
        *   Advance position: $t = t_{min} + i \cdot h + t_{offset}$.
        *   Sample volume at $\mathbf{p} = \mathbf{o} + t\boldsymbol{\omega}$:
            *   Get extinction $\sigma_t$ or opacity $\alpha_i$. If using density $\sigma$, compute $\alpha_i = 1 - e^{-\sigma_i h}$ [S3, unofficial][S7].
            *   Get emission $g_i$ (radiance added per unit length) [S6, S7].
        *   **Update Transmission:**
            *   $T_{curr} = T_{curr} \cdot (1 - \alpha_i)$ [S7].
            *   Store $T_{curr}$ if using back-to-front compositing [S3, unofficial].
        *   **Accumulate Color (Back-to-Front):**
            *   The contribution of the current sample to the final color is its emission weighted by the transmission of all *subsequent* (closer to viewer) samples, plus the attenuation of the background.
            *   Using the discrete form derived from the integrating factor solution [S7]:
                $$ L_{out} = L_{bg} \prod_{k=0}^{N-1} (1-\alpha_k) + \sum_{i=0}^{N-1} g_i \prod_{j=i+1}^{N-1} (1-\alpha_j) $$
            *   In a forward loop with a precomputed transmission array or running product from the back, this is implemented as:
                $$ C_{acc} = C_{acc} + g_i \cdot T_{after\_i} $$
                where $T_{after\_i}$ is the transmission from sample $i+1$ to the end.
            *   Alternatively, using front-to-back accumulation (common in real-time shaders):
                $$ C_{acc} = C_{acc} + (1 - \alpha_{acc}) \cdot g_i \cdot h \quad \text{(if } g_i \text{ is per-length)} $$
                $$ \alpha_{acc} = \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_i $$
                *Note: The front-to-back form approximates the integral. The back-to-front form is exact for the discrete Riemann sum if transmission is handled correctly [S7].*
        *   **Early Termination:**
            *   If $\alpha_{acc} > 0.98$ (or a threshold like 0.9), break loop [S2, unofficial].

3.  **Background Contribution:**
    *   If using front-to-back accumulation, add background:
        $$ C_{final} = C_{acc} + L_{bg} \cdot (1 - \alpha_{acc}) $$
    *   If using back-to-front, the background term $L_{bg} \prod (1-\alpha_k)$ is included in the sum or added at the end [S7].

4.  **Normalization:**
    *   If $\alpha_{acc} > 1.0$ (due to precision), clamp or normalize [S2, unofficial].
    *   Output $C_{final}$ with $\alpha_{out} = \min(\alpha_{acc}, 1.0)$.

**Pseudocode (Front-to-Back Accumulation):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def emission_absorption_ray_march(ray_origin, ray_direction, volume_sampler, step_size, max_steps, background_color):
    """
    Renders a volume using emission-absorption ray marching.
    Uses front-to-back accumulation for early termination.
    """
    # 1. Initialize
    accumulator_color = [0.0, 0.0, 0.0]
    accumulator_alpha = 0.0
    t = 0.0
    
    # Optional jitter
    t += random() * step_size
    
    # 2. March
    for i in range(max_steps):
        # Advance
        t += step_size
        if t >= 1.0: # Assuming normalized ray length 0..1
            break
            
        # Sample volume
        position = ray_origin + ray_direction * t
        density = volume_sampler.sample_density(position)
        emission = volume_sampler.sample_emission(position)
        
        # Compute opacity for this step
        # alpha = 1 - exp(-sigma_t * step_size)
        # Using approximation or exact:
        alpha = 1.0 - math.exp(-density * step_size)
        
        # Early termination check
        if accumulator_alpha > 0.98:
            break
            
        # Front-to-back compositing
        # Weight of current sample relative to what's left to see
        weight = (1.0 - accumulator_alpha) * alpha
        
        # Add emission contribution
        # emission is typically radiance per unit length, so multiply by step_size
        accumulator_color += weight * emission * step_size
        
        # Update accumulated alpha
        accumulator_alpha += weight
        
    # 3. Add background
    # The remaining transmission is (1 - accumulator_alpha)
    remaining_transmission = 1.0 - accumulator_alpha
    final_color = accumulator_color + background_color * remaining_transmission
    
    return final_color, min(accumulator_alpha, 1.0)
```

**Parameters:**
*   `step_size`: Distance between samples. Default 0.5 in some implementations [S4, unofficial]. Smaller values increase accuracy but cost more.
*   `max_steps`: Maximum iterations per ray. Default 2000 [S4, unofficial].
*   `uExtinction`: Extinction coefficient scaling factor [S2, unofficial].
*   `_AbsorptionCoefficient`: Used in Beer-Lambert law for light density [S5, unofficial].

**Edge Cases:**
*   **Precision:** If $\alpha_i$ is very small, $1-\alpha_i$ may suffer from floating-point cancellation. Use `expm1` or higher precision for transmission products [S3, unofficial].
*   **Unbounded Emission:** If absorption is zero, emission accumulates linearly without bound, potentially exceeding display range [S6].
*   **Degenerate Input:** Empty volume (zero density) should return background color immediately.

## Subtopics

### [Volumetric Liquid Rendering](4017-volumetric-liquid-rendering.md)
A specific application for liquids like lava lamp wax, masking visibility with surface depth and driving emission by proximity to lights.

### [Energy-based Volume Rendering (Energy Engine)](8218-energy-based-volume-rendering-energy-engine.md)
Decouples absorption and emission into separate ray buffers for precise control over attenuation of emitted light.

### Emission-only Volume Rendering
Renders glowing media by integrating the source term $g(s)$ along the ray, neglecting absorption [S6]. It is computationally simpler but lacks an upper bound on intensity, risking overflow [S6].

### Alpha Ray Marching
Computes opacity, transmission, and blend weights without accumulating final color [S3, unofficial]. Used for auxiliary losses or intermediate states. It calculates cumulative transmission with epsilon for stability and extracts background transmission [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of steps. Early termination reduces average cost for opaque regions [S7].
*   **Memory Complexity:** $O(1)$ for front-to-back accumulation. $O(N)$ if storing full transmission history for back-to-front compositing [S3, unofficial].
*   **Performance:** Deterministic and stable. Performance is limited by memory bandwidth when sampling 3D textures [S4, unofficial].
*   **Strengths:** Physically plausible for non-scattering media; simple to implement; supports early termination [S7].
*   **Limitations:** Ignores scattering (in-scattering/out-scattering), making it inaccurate for dense fog or smoke where light redirection is significant [S6]. Requires careful tuning of step size to avoid artifacts [S2, unofficial].

## Background
The method is derived from the Radiative Transfer Equation (RTE) for participating media [S5, unofficial][S6]. The specific differential equation for emission and absorption, $\frac{dI}{ds} = g(s) - \tau(s)I(s)$, is solved using an integrating factor [S6, S7]. This approach was formalized in direct volume rendering literature, notably by Levoy [S6], who distinguished it from emission-only and absorption-only models. It is widely used in medical imaging and scientific visualization for segmented data where tissue types have distinct emission and absorption properties [S6].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S3] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
[S4] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S5] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S6] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
[S7] Direct_Volume_Rendering.md, https://github.com/pyvista/pyvista/files/9562449/Direct_Volume_Rendering.md, fetched 2026-10-29
