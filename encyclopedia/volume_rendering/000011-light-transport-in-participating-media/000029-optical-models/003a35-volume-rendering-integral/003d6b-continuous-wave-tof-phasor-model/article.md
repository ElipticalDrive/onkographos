# Continuous-Wave ToF Phasor Model

Also called: C-ToF Phasor Model, Phasor Volume Rendering

## Overview
The Continuous-Wave ToF Phasor Model extends the volume rendering integral to simulate images captured by continuous-wave Time-of-Flight (C-ToF) cameras by incorporating a complex-valued path length importance function [S1, unofficial]. It differs from standard volume rendering by weighting contributions based on light travel distance and from depth-map-based methods by operating directly on complex phasor values, thereby avoiding wrap-around ambiguities and handling multi-path interference more robustly [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera origin $\mathbf{x}$, viewing direction $\boldsymbol{\omega}_0$, modulation frequency $f$, speed of light $c$, scattering coefficient $\sigma(\mathbf{x}_t)$, incident light intensity $I_s(\mathbf{x}_t, \boldsymbol{\omega}_0)$, and transmittance $T(\mathbf{x}, \mathbf{x}_t)$ along the ray [S1, unofficial].
*   **Output:** Complex-valued phasor radiance $L_{\text{ToF}}(\mathbf{x}, \boldsymbol{\omega}_0)$ [S1, unofficial].

**Data Structures:**
The output is a complex number, typically stored as two floating-point values representing the real and imaginary components, or as magnitude and phase [G].

**Formulas:**
The core rendering equation for a C-ToF camera is [S1, unofficial]:

$$
L_{\text{ToF}}(\mathbf{x}, \boldsymbol{\omega}_0) = \int_{t_n}^{t_f} \frac{T(\mathbf{x}, \mathbf{x}_t)^2}{\|\mathbf{x} - \mathbf{x}_t\|^2} \sigma(\mathbf{x}_t) I_s(\mathbf{x}_t, \boldsymbol{\omega}_0) W(2\|\mathbf{x} - \mathbf{x}_t\|) dt
$$

Where:
*   $t_n, t_f$: Near and far clipping distances along the ray [S1, unofficial].
*   $\mathbf{x}_t$: Position on the ray at parameter $t$, defined as $\mathbf{x}_t = \mathbf{x} + t\boldsymbol{\omega}_0$ [S1, unofficial].
*   $T(\mathbf{x}, \mathbf{x}_t)$: Transmittance from camera $\mathbf{x}$ to $\mathbf{x}_t$. The square $T^2$ accounts for attenuation on both the outgoing and return paths [S1, unofficial].
*   $\|\mathbf{x} - \mathbf{x}_t\|$: Euclidean distance between camera and scene point. When $\boldsymbol{\omega}_0$ is normalized, this distance equals $t$ [G]. The inverse-square term $\frac{1}{\|\mathbf{x} - \mathbf{x}_t\|^2}$ models the geometric falloff of a point source [S1, unofficial].
*   $W(d)$: Path length importance function for distance $d$. For C-ToF, light travels distance $d = 2\|\mathbf{x} - \mathbf{x}_t\|$ (round trip) [S1, unofficial].

The phasor function $W(d)$ is defined as [S1, unofficial]:

$$
W(d) = \exp\left(i \frac{2\pi d f}{c}\right)
$$

Substituting $d = 2\|\mathbf{x} - \mathbf{x}_t\|$:

$$
W(2\|\mathbf{x} - \mathbf{x}_t\|) = \exp\left(i \frac{4\pi f}{c} \|\mathbf{x} - \mathbf{x}_t\|\right)
$$

Where:
*   $f$: Modulation frequency of the C-ToF signal. Typical value: $30 \text{ MHz}$ [S1, unofficial].
*   $c$: Speed of light, $\approx 3 \cdot 10^8 \text{ m/s}$ [S1, unofficial].
*   $i$: Imaginary unit [S1, unofficial].

**Parameters:**
*   $f$: Modulation frequency. Typical value: $30 \text{ MHz}$ [S1, unofficial]. This sets the unambiguous range to $c/2f \approx 5 \text{ m}$ [S1, unofficial].

**Pseudocode:**
The following Python-like pseudocode computes the phasor radiance for a single ray. It assumes standard volume rendering integration is handled by the caller, and uses $t$ directly as the distance $\|\mathbf{x} - \mathbf{x}_t\|$ (valid when $\boldsymbol{\omega}_0$ is normalized).

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import cmath

def compute_c_to_f_phasing(distance, modulation_freq, speed_of_light):
    """
    Computes the complex phasor weight for a given one-way distance.
    
    Args:
        distance (float): One-way distance from camera to point (meters).
        modulation_freq (float): Modulation frequency f (Hz).
        speed_of_light (float): Speed of light c (m/s).
        
    Returns:
        complex: The phasor weight W(2*distance).
    """
    # Round-trip distance
    d = 2.0 * distance
    
    # Phase calculation: 2 * pi * f * d / c
    phase = 2.0 * math.pi * modulation_freq * d / speed_of_light
    
    # Complex exponential: exp(i * phase)
    return cmath.exp(1j * phase)

def render_c_to_f_ray(ray_origin, ray_direction, samples, modulation_freq=30e6, speed_of_light=3e8):
    """
    Renders a single ray using the C-ToF Phasor Model.
    
    Args:
        ray_origin (tuple): Camera position x (3D vector).
        ray_direction (tuple): Viewing direction omega_0 (3D vector, normalized).
        samples (list): List of tuples (t, sigma, I_s, T_prev) sorted by t.
        modulation_freq (float): C-ToF modulation frequency.
        speed_of_light (float): Speed of light.
        
    Returns:
        complex: The accumulated phasor radiance L_Tof.
    """
    # Initialize accumulator
    phasor_radiance = 0.0 + 0.0j
    
    # Small epsilon to avoid division by zero at the camera origin
    epsilon = 1e-6
    
    for t, sigma, I_s, T_prev in samples:
        # Distance from camera to point equals t for normalized direction
        dist = t
        
        # Guard against division by zero
        if dist < epsilon:
            dist = epsilon
            
        # Compute phasor weight for round-trip distance
        w = compute_c_to_f_phasing(dist, modulation_freq, speed_of_light)
        
        # Transmittance squared (outgoing and return path)
        T_squared = T_prev ** 2
        
        # Inverse square falloff
        inv_dist_sq = 1.0 / (dist ** 2)
        
        # Contribution: T^2 / dist^2 * sigma * I_s * W
        contribution = (T_squared * inv_dist_sq * sigma * I_s * w)
        
        # Accumulate
        phasor_radiance += contribution
        
    return phasor_radiance
```

**Edge Cases and Pitfalls:**
*   **Multi-path interference:** Multiple points along a ray contribute to the signal, resulting in a linear combination of phasor radiance values [S1, unofficial]. This degrades depth quality, causing 'flying pixel' artifacts around depth edges or with semi-transparent/specular objects [S1, unofficial].
*   **Wrap-around errors:** If deriving depth maps from phase, ranges $> c/2f$ are ambiguous [S1, unofficial]. Modeling raw phasor images avoids this [S1, unofficial].
*   **Noise:** Depth values become unreliable when reflected light is small [S1, unofficial]. Direct phasor modeling is more robust to sensor noise than derived depth maps [S1, unofficial].

## Subtopics

### [Collocated Point Light Source Model](10718-collocated-point-light-source-model.md)
This model defines the radiance contribution from an ideal Time-of-Flight (ToF) camera that responds exclusively to light emitted by an infrared (IR) point source collocated with the camera lens, ignoring all ambient illumination. It sets apart from general volume rendering by enforcing a strict geometric constraint where the incident light direction $\boldsymbol{\omega}_i$ is identical to the viewing direction $\boldsymbol{\omega}_0$ (backscattering along the same ray), and by incorporating the inverse-square falloff of a point source.

## Operating characteristics
*   **Time Complexity:** Same as standard volume rendering, $O(N)$ where $N$ is the number of samples per ray, plus the cost of complex arithmetic per sample [G].
*   **Memory Complexity:** $O(1)$ per ray for the accumulator, plus the memory required for the scene representation (e.g., NeRF weights, grid) [G].
*   **Strengths:** Avoids wrap-around errors for ranges $> c/2f$; robust to sensor noise in dark objects; handles multi-path interference at depth edges better than deriving depth maps [S1, unofficial].
*   **Limitations:** Multi-path interference can still cause 'flying pixel' artifacts; requires complex number arithmetic; unambiguous range is limited by modulation frequency $f$ (e.g., $\approx 5 \text{ m}$ for $30 \text{ MHz}$) [S1, unofficial].
*   **When to use:** When simulating C-ToF camera behavior directly, especially for scenes with large depth ranges or where multi-path interference is a concern [S1, unofficial].
*   **When not to use:** When only real-valued intensity or depth maps are needed and the scene depth is well within the unambiguous range, simpler models may suffice [S1, unofficial].

## Background
The method originates from the TöRF paper, which extends volume rendering to simulate Time-of-Flight cameras [S1, unofficial]. It builds on prior work in transient rendering frameworks [17, 39] and neural transient fields (NeTFs) [46] by incorporating a path length importance function [S1, unofficial]. The specific phasor model for C-ToF cameras is derived from the physics of continuous-wave modulation, where the phase shift of the reflected signal encodes distance [S1, unofficial].

## Sources
[S1] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
