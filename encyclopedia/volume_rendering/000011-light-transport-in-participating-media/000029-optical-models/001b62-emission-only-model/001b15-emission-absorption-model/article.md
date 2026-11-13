# Emission-Absorption Model

Also called: Absorption and Emission Model, Absorption and Emitter Model, Absorption-emission model, Density-Emitter Model, Max model, Max's Model, Williams and Max model

## Overview
The Emission-Absorption Model is the standard optical model for direct volume rendering, computing the final radiance reaching the eye by integrating light emitted by a participating medium along a ray, attenuated by the medium's absorption [S1, S2, S3, S8, unofficial]. It solves the problem of determining visible radiance from a 3D scalar field representing a participating medium by modeling the volume as filled with small light-emitting particles described by a density function, rather than modeling them individually [S2, unofficial][S12]. Unlike its parent [Emission-only Model](1016-emission-only-model.md), which only adds intensity, this model accounts for both light addition (emission) and light blocking (absorption/occlusion), allowing for the distinction of depth order along the ray [S19, unofficial]. It sets itself apart from full radiative transfer models by completely ignoring scattering, which maintains interactivity while providing a high level of realism for phenomena like clouds, smoke, and opaque surfaces in NeRFs [S3, unofficial][S17].

## How it works
**Inputs and Outputs:**
*   **Input:** A viewing ray $\mathcal{R}$ parameterized by distance $s$, entering the volume at $s_0$ and exiting at $D$ (or $s_n$) [S7, S8, S13, unofficial]. A scalar field or density function $\rho(s)$ (or voxel data) defining the medium [S2, unofficial][S12]. Transfer functions mapping scalar values to extinction coefficient $\tau(s)$ (or $\kappa(s)$) and source term $g(s)$ (or $q(s)$, $C(s)$) [S7, S8, S13, unofficial]. Background radiance $I_0$ at $s_0$ [S7, S8, S13, unofficial].
*   **Output:** Final radiance $I(D)$ or color $C(r)$ at the camera [S6][S8, unofficial].

**Data Structures:**
The method operates on discrete samples along the ray [S5][S8, S10, unofficial]. For a ray of length $L$ (or $D-s_0$) divided into $n$ segments of length $\Delta s$ (or $\Delta t$, $\Delta x$):
*   **Sample $i$:** Position $s_i = s_0 + i \cdot \Delta s$ [S8, S10, unofficial].
*   **Extinction/Absorption Coefficient:** $\tau(s_i)$ or $\kappa(s_i)$ [S7, S9, unofficial].
*   **Source/Emission Term:** $g(s_i)$, $q(s_i)$, or $C(s_i)$ [S7, S8, S10, unofficial].
*   **Transmittance of segment $i$:** $T_i = e^{-\tau(s_i) \Delta s}$ [S7, S8, unofficial].
*   **Opacity of segment $i$:** $\alpha_i = 1 - T_i$ [S10, unofficial].

**Formulas:**
The continuous integral form of the volume rendering equation is given by [S2, S7, S8, S9, unofficial][S12]:
$$ I(s) = I(s_0) e^{-\int_{s_0}^s \tau(t) \, dt} + \int_{s_0}^s g(\tilde{s}) e^{-\int_{\tilde{s}}^s \tau(t) \, dt} \, d\tilde{s} $$
Where:
*   $I(s_0)$ is the incoming radiance (background) at the start of the ray [S7, S8, S13, unofficial].
*   $\tau(t)$ is the extinction (absorption) coefficient at point $t$ [S7, S8, unofficial].
*   $g(\tilde{s})$ is the emission (source) term at point $\tilde{s}$ [S7, S8, unofficial].
*   The first term represents background light attenuated by the volume [S7, S8, S13, unofficial].
*   The second term represents light emitted by the volume, attenuated by the volume between the emission point and the eye [S7, S8, S13, unofficial].

The optical depth $\tau(s_1, s_2)$ is defined as [S2, S9, unofficial][S11, unofficial][S12]:
$$ \tau(s_1, s_2) = \int_{s_1}^{s_2} \kappa(s) \, ds $$

**Discretization (Riemann Sum):**
The integral is approximated numerically using a Riemann sum by subdividing the interval $[s_0, D]$ into $n$ equal segments of length $\Delta s = (D-s_0)/n$ [S7, S8, S10, unofficial]. The discrete approximation is [S8, unofficial][S15]:
$$ I(D) \approx I_0 \prod_{i=0}^n e^{-\tau(s_i)\Delta s} + \sum_{i=0}^n g(s_i) \prod_{j=i+1}^n e^{-\tau(s_j)\Delta s} $$
Where $T_i = e^{-\tau(s_i)\Delta s}$ is the transmittance of the $i$-th segment [S7, S8, unofficial].

**Front-to-Back Compositing (Iterative Solution):**
A common implementation uses front-to-back compositing, accumulating color and opacity iteratively [S10, S19, unofficial]. Let $\hat{C}_i$ be the accumulated pre-multiplied color and $\alpha_i$ be the accumulated opacity after processing sample $i$ [S10, unofficial].
Initialize: $\hat{C}_{-1} = 0$, $\alpha_{-1} = 0$ [S20, unofficial].
For each sample $i$ from $0$ to $n-1$ (front to back):
1.  Calculate segment opacity: $\alpha_i = 1 - e^{-\tau(s_i)\Delta s}$ [S10, unofficial].
2.  Calculate segment pre-multiplied color: $\hat{C}_{seg, i} = g(s_i) \Delta s \cdot \alpha_i$ (if $g$ is intensity per unit length) or simply $g(s_i)$ if $g$ already includes the $\Delta s$ and $\tau$ scaling (see Density-Emitter specifics below) [S10, unofficial][S15]. Note: In standard alpha compositing, the contribution is often defined as $c_i = C(s_i) \alpha_i$ where $C$ is the color [S4, S10, unofficial].
3.  Update accumulated color: $\hat{C}_i = \hat{C}_{i-1} + (1 - \alpha_{i-1}) \hat{C}_{seg, i}$ [S10, unofficial].
4.  Update accumulated opacity: $\alpha_i = \alpha_{i-1} + (1 - \alpha_{i-1}) \alpha_i$ [S10, unofficial].
5.  **Early Ray Termination:** If $\alpha_i \ge 1$ (or sufficiently close to 1), stop processing further samples for this ray [S10, unofficial].

**Density-Emitter Model Specifics:**
In the Density-Emitter Model (a specific case of Emission-Absorption), the source term $q(s)$ and absorption coefficient $\kappa(s)$ are proportional to particle density $\rho(s)$ [S9, unofficial][S12]:
$$ \kappa(s) = \kappa_0 \cdot \rho(s) \quad \text{and} \quad q(s) = q_0 \cdot \rho(s) $$
Where $\kappa_0$ and $q_0$ are constants [S9, unofficial].
The emission contribution $b_k$ for a segment $[s_{k-1}, s_k]$ becomes [S9, unofficial]:
$$ b_k = \frac{q_0}{\kappa_0} (1 - \vartheta_k) $$
Where $\vartheta_k = e^{-\tau(s_{k-1}, s_k)}$ is the transparency of the segment [S9, unofficial].
The iterative update is [S9, unofficial]:
$$ I(s_k) = I(s_{k-1}) \cdot \vartheta_k + \frac{q_0}{\kappa_0} (1 - \vartheta_k) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_emission_absorption(ray_origin, ray_direction, volume_data, transfer_function, num_samples, t_start, t_end, background_color):
    """
    Renders a single ray using the Emission-Absorption model with front-to-back compositing.
    
    Args:
        ray_origin: list of 3 floats, origin of the ray.
        ray_direction: list of 3 floats, normalized direction of the ray.
        volume_data: function(pos) -> float, samples density at a point.
        transfer_function: function(density) -> (extinction, color), maps density to optical properties.
        num_samples: int, number of samples along the ray.
        t_start: float, start parameter of the ray segment inside the volume.
        t_end: float, end parameter of the ray segment inside the volume.
        background_color: list of 3 floats, background radiance.
        
    Returns:
        list of 3 floats, the final color radiance.
    """
    ray_length = t_end - t_start
    if ray_length <= 0:
        return list(background_color)
    
    delta_s = ray_length / num_samples
    
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    for i in range(num_samples):
        t = t_start + i * delta_s
        pos = [ray_origin[j] + ray_direction[j] * t for j in range(3)]
        
        density = volume_data(pos)
        extinction, color = transfer_function(density)
        
        tau = extinction * delta_s
        opacity_segment = 1.0 - math.exp(-tau)
        
        if accumulated_opacity >= 1.0:
            break
            
        pre_multiplied_color = [color[j] * opacity_segment for j in range(3)]
        
        weight = 1.0 - accumulated_opacity
        accumulated_color = [accumulated_color[j] + weight * pre_multiplied_color[j] for j in range(3)]
        accumulated_opacity = accumulated_opacity + weight * opacity_segment
        
    final_color = [accumulated_color[j] + (1.0 - accumulated_opacity) * background_color[j] for j in range(3)]
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Optical Depth Extremes:** If optical depth $\tau$ is infinitely large, transparency becomes 0. If $\tau$ is 0, transparency is 1 [S2, unofficial].
*   **Scattering Neglect:** The model fails if scattering is significant, as it changes the direction and frequency of radiant energy [S9, unofficial][S17].
*   **Aliasing:** Pre-classification and second-order integration can cause aliasing artifacts, especially when grid variables vary significantly between neighboring cells [S11, unofficial].
*   **Ordering:** Compositing requires strict ordering (back-to-front or front-to-back) because the equations are not commutative [S15].
*   **Thickness Variation:** Constant transfer functions cause color variation based on ray thickness through an isosurface (darker/brighter) because rays accumulate different amounts of emission [S11, unofficial].

## Subtopics

### [Absorption Term Restriction](1521-absorption-term-restriction.md)
Restricts the model to absorption only, removing emission and scattering to reduce variance in specific spectral tracking contexts.

### [Homogeneous Media](1813-homogeneous-media.md)
Assumes constant density, enabling closed-form transmittance calculations.

### [Particle Model](2005-particle-model.md)
A special case where $g(s) = C(s) \tau(s)$, assuming identical spherical particles and constant color $C$ along the ray.

### [Point-Sampled Spectrum Model](3468-point-sampled-spectrum-model.md)
Discrete spectral representation using uniform bands, supporting Doppler effect but suffering from information loss.

### Independent Source Model
A variant where the source term $g(s)$ is defined independently of the extinction coefficient $\tau(s)$, allowing for transparent glowing gas (non-zero $g$ when $\tau$ is zero) [S14, unofficial]. Unlike the particle model where interpolated $g(s)$ is quadratic on a ray segment (if $f$ is linear), an independently defined and interpolated $g(s)$ is linear [S14, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of samples per ray and $R$ is the number of rays (pixels) [S10, S19, unofficial]. Early ray termination can reduce effective $N$ for opaque regions [S10, unofficial].
*   **Memory Complexity:** $O(V)$ for storing the volume data, where $V$ is the number of voxels [S10, unofficial].
*   **Performance:** Computationally tractable and suitable for interactive rendering, unlike full scattering models [S3, S10, unofficial][S17]. Performance is heavily dependent on the efficiency of volume traversal and interpolation [S8, unofficial].
*   **Strengths:** High realism for participating media like smoke and clouds; simple to implement; supports early ray termination; forms the basis for NeRFs [S3, unofficial][S17].
*   **Limitations:** Cannot model shadows cast by the volume onto itself or other objects (no in-scattering); inaccurate for media with significant scattering (e.g., fog with strong forward scattering, milk) [S9, unofficial][S17].
*   **When to Use:** Interactive visualization of medical data, scientific simulations, and neural radiance fields where scattering is negligible or approximated [S7, S13, unofficial][S17].
*   **When Not to Use:** Physically accurate rendering of participating media where scattering plays a major role (e.g., realistic fog, subsurface scattering effects) [S9, unofficial].

## Background
The Emission-Absorption Model is based on Boltzmann's transport equation but completely ignores scattering [S4, unofficial]. It was formalized by Nelson Max and Williams in the early 1990s (Max 1995, Williams and Max 1992) [S7, S19, unofficial]. It is the most widely used optical rendering model for volume rendering [S8, unofficial]. It is traditionally used in medical and scientific visualization [S7, unofficial] and is the specific light transport model underpinning neural radiance fields (NeRFs) [S17]. The model simplifies the full Radiative Transfer Equation (RTE) by considering only emission and absorption/extinction [S17].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S3] Direct Volume Rendering Data, https://www.scribd.com/document/1050958395/Direct-Volume-Rendering-Data, fetched 2026-10-25 (unofficial)
[S4] Volume Ray Casting Techniques, https://www.scribd.com/document/102700078/03-raycasting, fetched 2026-10-25 (unofficial)
[S5] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
[S6] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S7] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S8] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S9] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S10] Volume Rendering with WebGL, https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl, fetched 2026-10-26 (unofficial)
[S11] The physics of volume rendering, https://arxiv.org/abs/1410.6022v1, fetched 2026-10-26 (unofficial)
[S12] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26
[S13] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S14] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30 (unofficial)
[S15] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27
[S16] PDF Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27 (book)
[S17] Volume Rendering Integral: Definition & AI/ML Use, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/neural-scene-representations/volume-rendering-integral, fetched 2026-10-28
[S18] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S19] Direct Volume Rendering, https://cscheid.net/writing/data_visualization/direct_volume_rendering/, fetched 2026-10-29 (unofficial)
