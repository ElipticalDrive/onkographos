# Emission-Absorption Model

Also called: Williams and Max model, Absorption-emission model, Attenuated emission model, Absorption and Emission Model

## Overview
The Emission-Absorption Model computes the final radiance reaching the eye by integrating the light emitted by a participating medium along a viewing ray, attenuated by the medium's absorption coefficient [S1, S4, S5, unofficial]. It solves the problem of determining visible radiance from a 3D scalar field representing a participating medium, serving as the fundamental equation for direct volume rendering [S5, S12, unofficial]. Unlike the [Emission-only Model](1016-emission-only-model.md), which adds intensity without attenuation, this model accounts for occlusion, allowing the distinction of depth order along the ray [S14, unofficial]. It sets itself apart from scattering models by neglecting in-scattering (light bouncing into the ray from other directions), making it computationally tractable for interactive applications while still simulating phenomena like clouds, smoke, and opaque surfaces in NeRFs [S12, S13, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field $f(X)$ representing volumetric data (e.g., density), a ray $\mathcal{R}$ parameterized by distance $t$ from the camera origin $s_0$ to the far boundary $D$, and transfer functions mapping $f$ to optical properties.
*   **Output:** The final radiance $I(D)$ (or color $C(r)$) at the pixel corresponding to the ray.

**Data Structures:**
The scalar field is typically stored as a 3D texture or grid. For discrete integration, the ray is subdivided into $n$ segments of length $\Delta t = (D-s_0)/n$ [S3, S4, unofficial]. Samples are taken at positions $s_i = s_0 + i \Delta t$.

**Formulas:**
The continuous integral form of the radiative transfer equation, neglecting scattering, is [S1, unofficial][S7][S9, unofficial]:

$$ I(s_n) = I(s_0) e^{-\int_{s_0}^{s_n} \tau(t) \, dt} + \int_{s_0}^{s_n} g(\tilde{s}) e^{-\int_{\tilde{s}}^{s_n} \tau(t) \, dt} \, d\tilde{s} $$

Where:
*   $I(s_0)$ is the background radiance entering the volume at $s_0$.
*   $I(s_n)$ is the final radiance reaching the eye.
*   $\tau(t)$ is the extinction (absorption) coefficient at point $t$.
*   $g(\tilde{s})$ is the source term (emission) at point $\tilde{s}$.
*   The term $e^{-\int_{s_0}^{s} \tau(t) \, dt}$ is the transmittance $T(s_0, s)$, representing the fraction of light that survives absorption from $s_0$ to $s$ [S3, S13, unofficial].

**Discretization:**
The integral is approximated using a Riemann sum [S3, S4, unofficial]. The ray is divided into $n$ segments. The transmittance of the $i$-th segment is $T_i = e^{-\tau(s_i) \Delta t}$ [S3, unofficial]. The emission contribution of the segment is $C_i = g(s_i) \Delta t$ (or simply $g(s_i)$ if $\Delta t$ is absorbed into the source term definition) [S3, S6, unofficial].

The discrete back-to-front summation is [S7][S11, book]:

$$ I(D) \approx I_0 \prod_{i=0}^n T_i + \sum_{i=0}^n C_i \prod_{j=i+1}^n T_j $$

Alternatively, using front-to-back compositing with pre-multiplied opacity $\hat{C}_i = C_i \alpha_i$ and segment opacity $\alpha_i = 1 - T_i$ [S6, unofficial]:

$$ \hat{C}_{accum} = \hat{C}_{accum} + (1 - \alpha_{accum}) \hat{C}_i $$
$$ \alpha_{accum} = \alpha_{accum} + (1 - \alpha_{accum}) \alpha_i $$

**Parameters:**
*   $n$: Number of sample points along the ray. Typical values depend on resolution and required accuracy; higher $n$ reduces aliasing but increases cost [S3, S4, unofficial].
*   $\Delta t$: Segment length, $\Delta t = (D-s_0)/n$ [S3, unofficial].
*   Transfer Functions: Map scalar field values to $\tau$ and $g$. Commonly Gaussian or piecewise linear functions [S7].

**Pseudocode:**
The following implements the front-to-back compositing approach, which allows for early ray termination [S6, unofficial][S7].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_emission_absorption(ray_origin, ray_direction, volume_grid, transfer_functions, max_distance):
    """
    Renders a single ray using the Emission-Absorption model.
    
    Args:
        ray_origin: Vector3, origin of the ray.
        ray_direction: Vector3, normalized direction of the ray.
        volume_grid: 3D array or texture providing scalar values f(x,y,z).
        transfer_functions: Object providing map_to_extinction(f) and map_to_emission(f).
        max_distance: Float, distance to the far boundary of the volume.
        
    Returns:
        Vector4: Final color (RGB) and accumulated opacity (A).
    """
    # Initialize accumulated color and opacity
    # Using pre-multiplied alpha: color is already multiplied by alpha
    acc_color = Vector4(0.0, 0.0, 0.0, 0.0)
    acc_alpha = 0.0
    
    # Determine step size. For simplicity, assume fixed step count N or adaptive.
    # Here we use a fixed number of steps N for clarity, though adaptive is common.
    N = 100 
    delta_t = max_distance / N
    
    # Current position along the ray
    t = 0.0
    
    for i in range(N):
        # Early ray termination: if the ray is fully opaque, stop
        if acc_alpha >= 1.0:
            break
            
        # Sample the scalar field at the current position
        # Interpolation (e.g., trilinear) is assumed to be handled by volume_grid.sample()
        scalar_value = volume_grid.sample(ray_origin + ray_direction * t)
        
        # Map scalar value to optical properties using transfer functions
        extinction = transfer_functions.map_to_extinction(scalar_value)
        emission = transfer_functions.map_to_emission(scalar_value)
        
        # Calculate segment transmittance and opacity
        # T_i = exp(-extinction * delta_t)
        # alpha_i = 1 - T_i
        # Optimization: approximate alpha_i = extinction * delta_t for small values [S3, unofficial]
        segment_alpha = 1.0 - math.exp(-extinction * delta_t)
        
        # Pre-multiplied emission color for this segment
        # C_i = emission * segment_alpha (if emission is defined as radiance per unit opacity)
        # Or simply emission if emission is defined as radiance contribution directly.
        # Based on [S6, unofficial], C(i*ds) is emitted color, and we use pre-multiplied opacity.
        # Let's assume emission() returns the color contribution C(s) such that
        # the term in the integral is C(s) * mu(s). 
        # Standard DVR often defines source term g(s) = C(s) * mu(s).
        # Here we assume transfer_functions.map_to_emission returns the pre-multiplied color 
        # contribution for the segment, i.e., C_seg = emission_color * segment_alpha.
        # If emission() returns raw color, we must multiply by segment_alpha.
        # Let's assume emission() returns the color C(s) and we multiply by alpha.
        seg_color = emission * segment_alpha
        
        # Front-to-back compositing
        # acc_color_new = acc_color + (1 - acc_alpha) * seg_color
        # acc_alpha_new = acc_alpha + (1 - acc_alpha) * segment_alpha
        
        remaining_transparency = 1.0 - acc_alpha
        acc_color = acc_color + remaining_transparency * seg_color
        acc_alpha = acc_alpha + remaining_transparency * segment_alpha
        
        # Advance along the ray
        t += delta_t
        
    return acc_color, acc_alpha
```

**Edge Cases and Pitfalls:**
*   **Scattering:** The model explicitly neglects in-scattering. It cannot model light bouncing into the ray from other directions, such as the glow of a fog bank illuminated from behind by a sun that is not directly visible [S2][S12, unofficial].
*   **Aliasing:** Pre-classification (applying transfer functions before interpolation) can cause aliasing artifacts, especially with second-order integration [S7]. Post-classification (interpolating scalar data first, then applying transfer functions) reduces aliasing but is more memory-intensive [S7].
*   **Varying Thickness:** Using constant transfer functions for isosurfaces can cause color variation if the spatial thickness of the isosurface varies, as rays pick up different amounts of emission [S7].
*   **Compositing Order:** The equations are not commutative. Strict ordering (back-to-front or front-to-back) is required [S11, book].

## Subtopics

### [Homogeneous Media](1813-homogeneous-media.md)
Homogeneous Media is a specific optical model for participating media where volumetric properties (density, absorption, scattering) are constant throughout a defined geometric boundary. It simplifies the general Emission-Absorption Model by removing spatial variation, allowing for closed-form transmittance calculations. Unlike the Particle Model or Point-Sampled Spectrum Model, which handle discrete or sparse data, this method assumes a continuous, uniform medium.

### [Particle Model](2005-particle-model.md)
The Particle Model is a special case of the Emission-Absorption Model where the source term is defined as $g(s) = C(s) \tau(s)$, based on the assumption of identical spherical particles. It solves the volume rendering integral by simplifying it to a simple compositing equation when the color $C$ is constant along the ray. It sets itself apart from the general Emission-Absorption Model by enforcing a specific physical relationship between emission and absorption, and from the Point-Sampled Spectrum Model by assuming a continuous density field rather than discrete samples.

### [Point-Sampled Spectrum Model](3468-point-sampled-spectrum-model.md)
The Point-Sampled Spectrum Model is a discrete spectral representation method for volume rendering that approximates spectral distribution functions by dividing the spectrum into $N$ uniform bands via point-sampling at uniform intervals. It solves the problem of integrating emission and absorption along viewing rays in participating media with spectral variation, specifically supporting the Doppler effect. Unlike the [Piecewise Linear Spectrum Model](4709-piecewise-linear-spectrum-model.md), which uses continuous trapezoidal bands to ensure energy conservation and accurate Doppler shift representation, this model suffers from information loss and accuracy issues due to its fixed-band discretization.

### Absorption Term Restriction
An optical model restriction to the absorption term of the volume density model [S15]. It uses a 3D texture map containing the integral of the absorption coefficient along the viewing ray segment [S15].

### Independent Source Model
A variant where the source term $g(s)$ is defined independently of the extinction coefficient $\tau(s)$, allowing for transparent glowing gas (non-zero $g$ when $\tau$ is zero) [S9, unofficial]. Unlike the particle model where interpolated $g(s)$ is quadratic on a ray segment (if $f$ is linear), an independently defined and interpolated $g(s)$ is linear [S9, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples along the ray. The total complexity is $O(P \cdot N)$ for $P$ pixels.
*   **Memory Complexity:** $O(V)$ for the volume grid, where $V$ is the number of voxels.
*   **Performance:** Computationally efficient compared to full scattering models, making it suitable for real-time rendering on GPUs [S4, S6, unofficial]. However, high-resolution data sets can still be challenging for general-purpose CPUs [S4, unofficial].
*   **Strengths:** Simple to implement, fast, handles occlusion and depth ordering correctly, sufficient for many phenomena like clouds and smoke [S12, S13, unofficial].
*   **Limitations:** Cannot model scattering effects (e.g., light bending, glow from indirect illumination), which may lead to unrealistic appearances in dense media [S2][S5, unofficial].
*   **When to Use:** Interactive visualization, medical imaging, scientific visualization where scattering is negligible or approximated [S3, S9, unofficial].
*   **When Not to Use:** Photorealistic rendering requiring global illumination effects within the volume, or scenes with significant participating media scattering [S2][S5, unofficial].

## Background
The Emission-Absorption Model is the simplest optical model for volume rendering, traditionally used in medical and scientific visualization [S3, S4, unofficial]. It follows Nelson Max's exposition and is often referred to as the Williams and Max model [S14, unofficial]. The model simplifies the full Radiative Transfer Equation (RTE) by considering only emission and absorption/extinction, neglecting scattering [S12, unofficial]. This simplification is computationally tractable and sufficient for modeling many phenomena, including the appearance of clouds, smoke, and the opaque surfaces represented in Neural Radiance Fields (NeRFs) [S12, unofficial]. The model was popularized by Williams and Max in the early 1990s [S3, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S3] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S4] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S5] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S6] Volume Rendering with WebGL, https://www.willusher.io/webgl/2019/01/13/volume-rendering-with-webgl, fetched 2026-10-26 (unofficial)
[S7] The physics of volume rendering, https://arxiv.org/abs/1410.6022v1, fetched 2026-10-26
[S8] [PDF] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S9] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S10] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2022/homework2.pdf, fetched 2026-10-30
[S11] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S12] Volume Rendering Integral: Definition & AI/ML Use, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/neural-scene-representations/volume-rendering-integral, fetched 2026-10-28 (unofficial)
[S13] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S14] Direct Volume Rendering, https://cscheid.net/writing/data_visualization/direct_volume_rendering/, fetched 2026-10-29 (unofficial)
[S15] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
