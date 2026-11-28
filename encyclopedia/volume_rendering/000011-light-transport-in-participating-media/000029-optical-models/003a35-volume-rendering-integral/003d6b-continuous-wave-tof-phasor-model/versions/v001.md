# Continuous-Wave ToF Phasor Model

Also called: C-ToF Phasor Model, Phasor Volume Rendering

## Overview
The Continuous-Wave ToF Phasor Model extends the volume rendering integral to simulate images captured by continuous-wave time-of-flight (C-ToF) cameras by incorporating a complex-valued path length importance function $W(d)$ that weights contributions based on light travel distance [S1, unofficial]. Unlike methods that derive scalar depth maps from phase measurements, this model outputs a complex-valued phasor image, which avoids wrap-around ambiguities for ranges exceeding the unambiguous limit ($c/2f$), reduces noise in low-signal regions, and provides a more robust representation for handling multi-path interference at depth edges [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position $\mathbf{x}$, viewing direction $\boldsymbol{\omega}_0$, modulation frequency $f$, speed of light $c$, scene scattering coefficient $\sigma(\mathbf{x}_t)$, incident light intensity $I_s(\mathbf{x}_t, \boldsymbol{\omega}_0)$, and transmittance $T(\mathbf{x}, \mathbf{x}_t)$ along the ray.
*   **Outputs:** A complex-valued phasor radiance $L_{\text{ToF}}(\mathbf{x}, \boldsymbol{\omega}_0)$.

**Data Structures:**
The output is a complex number. In implementation, this is typically stored as two floating-point values (real and imaginary parts) or a magnitude-phase pair. The transmittance $T$ is usually precomputed or accumulated during ray marching.

**Formulas:**
The core rendering equation is defined as [S1, unofficial]:

$$
L_{\text{ToF}}(\mathbf{x}, \boldsymbol{\omega}_0) = \int_{t_n}^{t_f} \frac{T(\mathbf{x}, \mathbf{x}_t)^2}{\|\mathbf{x} - \mathbf{x}_t\|^2} \sigma(\mathbf{x}_t) I_s(\mathbf{x}_t, \boldsymbol{\omega}_0) W(2\|\mathbf{x} - \mathbf{x}_t\|) dt
$$

Where:
*   $t_n, t_f$: Near and far clipping planes along the ray.
*   $\mathbf{x}_t$: A point on the ray at parameter $t$.
*   $\|\mathbf{x} - \mathbf{x}_t\|$: Euclidean distance from the camera origin $\mathbf{x}$ to the scene point $\mathbf{x}_t$.
*   $W(d)$: The path length importance function for distance $d$.

The importance function for a C-ToF camera is a complex exponential (phasor) [S1, unofficial]:

$$
W(d) = \exp\left(i \frac{2\pi d f}{c}\right)
$$

Substituting $d = 2\|\mathbf{x} - \mathbf{x}_t\|$ (since light travels to the surface and back):

$$
W(2\|\mathbf{x} - \mathbf{x}_t\|) = \exp\left(i \frac{4\pi f}{c} \|\mathbf{x} - \mathbf{x}_t\|\right)
$$

**Parameters:**
*   $f$: Modulation frequency of the C-ToF camera signal. Typical value: 30 MHz [S1, unofficial].
*   $c$: Speed of light, approximately $3 \cdot 10^8$ m/s [S1, unofficial].

**Algorithm:**
1.  For each pixel, cast a ray from camera origin $\mathbf{x}$ in direction $\boldsymbol{\omega}_0$.
2.  Initialize complex accumulator $L_{\text{ToF}} = 0$.
3.  March along the ray from $t_n$ to $t_f$ with step size $\Delta t$.
4.  At each sample point $\mathbf{x}_t$:
    a. Calculate distance $d_{\text{cam}} = \|\mathbf{x} - \mathbf{x}_t\|$.
    b. Calculate round-trip distance $d = 2 d_{\text{cam}}$.
    c. Compute phasor weight $W = \exp(i \cdot 2\pi \cdot f \cdot d / c)$.
    d. Compute transmittance $T(\mathbf{x}, \mathbf{x}_t)$ from previous samples.
    e. Compute incident light $I_s(\mathbf{x}_t, \boldsymbol{\omega}_0)$ from the source.
    f. Compute scattering coefficient $\sigma(\mathbf{x}_t)$.
    g. Compute term: $\text{term} = \frac{T^2}{d_{\text{cam}}^2} \cdot \sigma \cdot I_s \cdot W$.
    h. Add to accumulator: $L_{\text{ToF}} \leftarrow L_{\text{ToF}} + \text{term} \cdot \Delta t$.
5.  Return $L_{\text{ToF}}$.

**Edge Cases and Pitfalls:**
*   **Multi-path Interference:** Multiple points along a ray contribute to the signal, resulting in a linear combination of phasor radiance values. This degrades depth quality, causing 'flying pixel' artifacts around depth edges or with semi-transparent/specular objects [S1, unofficial].
*   **Singularity:** The term $1/\|\mathbf{x} - \mathbf{x}_t\|^2$ approaches infinity as $\mathbf{x}_t \to \mathbf{x}$. In practice, the near clip plane $t_n$ prevents sampling at the camera origin.
*   **Precision:** Complex arithmetic must be handled carefully to avoid precision loss in the phase calculation for large distances.

## Subtopics

### [Collocated Point Light Source Model](10718-collocated-point-light-source-model.md)
Models radiance for an ideal ToF camera responding only to light from a collocated IR point source, ignoring ambient illumination.

## Operating characteristics
*   **Complexity:** Time complexity is linear with respect to the number of samples per ray, similar to standard volume rendering, but with the overhead of complex number arithmetic. Memory complexity is dominated by the scene representation (e.g., NeRF weights or voxel grid).
*   **Strengths:** Modeling raw phasor images avoids wrap-around errors for ranges $> c/2f$ (e.g., $>5$ m at 30 MHz), reduces noise in dark objects (weak signals), and handles multi-path interference at depth edges better than deriving depth maps [S1, unofficial].
*   **Limitations:** The output is complex-valued, requiring downstream processing (e.g., magnitude/phase extraction) for visualization or standard depth-based applications. Multi-path interference can still cause artifacts if not explicitly modeled or mitigated.
*   **When to use:** When simulating C-ToF camera behavior for dynamic scenes, especially when depth ambiguity or multi-path effects are significant.
*   **When not to use:** When only standard RGB rendering is needed, or when the scene depth is strictly within the unambiguous range and multi-path effects are negligible.

## Background
The method originates from the extension of volume rendering to time-of-flight cameras, specifically continuous-wave ToF sensors. It builds on the volume rendering integral and incorporates path length importance functions similar to those used in transient rendering frameworks [S1, unofficial]. The specific phasor formulation for C-ToF cameras was described by Pediredla et al. [39] and applied in the context of radiance fields by TöRF [S1, unofficial].

## Sources
[S1] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
