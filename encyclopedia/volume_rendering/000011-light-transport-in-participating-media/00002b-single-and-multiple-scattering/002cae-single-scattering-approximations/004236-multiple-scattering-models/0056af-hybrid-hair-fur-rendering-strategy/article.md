# Hybrid Hair/Fur Rendering Strategy

Also called: Hybrid Hair/Fur Rendering with Volumetric Multiple Scattering Approximation, Hybrid Rendering Strategy for Hair/Fur, Hybrid light transport estimator, Volumetric Approximation for Hair/Fur Multiple Scattering

## Overview
This method approximates multiple scattering in dense hair and fur by combining explicit geometric rendering for direct illumination with a volumetric participating medium model for higher-order scattering [S1]. It solves the computational bottleneck of simulating dozens of light bounces between fibers by treating the aggregate hair volume as an anisotropic medium, preserving high-frequency strand details while efficiently capturing soft volumetric glow [S1]. Unlike microflake models that assume opaque particles, this approach normalizes the hair BCSDF to define the phase function, preserving complex transmission paths [S1].

## How it works
The method employs a hybrid geometric–volumetric representation [S1]. Explicit fiber geometry handles direct lighting and first-bounce interactions to preserve fine details, while an equivalent anisotropic participating medium approximates the low-frequency, multi-bounce transport [S1].

### Inputs and Outputs
*   **Inputs:** Hair/fur geometry (strands), hair scattering model (BCSDF), lighting environment, camera parameters [S1].
*   **Outputs:** Rendered image with direct and multiple scattering contributions [S1].

### Data Structures
1.  **Volumetric Medium:** Derived from voxelized hair geometry [S1].
    *   **Fields:** Local fiber density, average fiber orientation (tangent), extinction coefficient $\sigma_t$, absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, and anisotropic phase function $P$ [S1].
    *   **Layout:** Heterogeneous grid where properties vary per voxel based on local strand density and orientation [S1].
2.  **Albedo Map:** Precomputed 1D function of incident inclination angle $\theta_i$ relative to local fiber direction, derived from the hair BCSDF [S1].

### Parameters
*   $N$: Number of octaves in the multiple scattering summation [S1].
*   $a$: Scalar controlling effective attenuation, allowing deeper light penetration for higher-order terms [S1].
*   $\gamma$: Scalar gain in $[0, 2]$ to moderate excessive color saturation [S1].
*   $c$: Parameter in $[0, 1]$ controlling the rate at which higher-order scattering becomes isotropic [S1].
*   **Base Model:** Yan et al. [2017a] double-cylinder fur model (accounts for cuticle, cortex, medulla) [S1].

### Steps

1.  **Precomputation (Voxelization):**
    Voxelize the hair geometry to compute local fiber densities and average orientations [S1]. Derive the equivalent volumetric medium properties ($\sigma_t, \sigma_s, \sigma_a, P$) from the hair fiber distribution [S1]. The albedo is precomputed as a function of the incident inclination angle $\theta_i$ [S1].

2.  **Rasterization Pass:**
    Accumulate per-pixel visibility and hair density in screen space [S1].

3.  **First Geometric Interaction (Direct Lighting):**
    Trace camera rays to find the first intersection with explicit hair geometry [S1]. Compute direct illumination using an arbitrary hair BCSDF and standard next-event estimation [S1]. Ensure the same scattering model is used consistently for the subsequent volumetric representation [S1].

4.  **Transition to Medium:**
    Sample an outgoing direction from the hair BCSDF at the first hit [S1]. Propagate the light path into the surrounding hair volume [S1]. Restrict geometric interactions to at most one; all subsequent transport is handled exclusively in the equivalent participating medium [S1].

5.  **Single Scattering in Medium:**
    *   Sample a scattering point $p$ along the view ray using delta tracking in the heterogeneous medium [S1].
    *   Select a random light direction $\omega_i$ [S1].
    *   March along $\omega_i$ to evaluate transmittance and direct illumination at $p$ [S1].
    *   Evaluate the transmittance $T$ along the shadow ray using ray marching through the medium:
        $$ T = \exp \left( -\int_0^t \sigma_t(x(s), \omega_i) ds \right) $$
    *   This yields the single-scattering estimate $L_{single}$ [S1].

6.  **Multiple Scattering Approximation:**
    Reweight the single-scattering estimate to approximate higher-order scattering [S1]. The total volumetric contribution $L$ is a sum of $N$ octaves:
    $$ L = \sum_{i=0}^{N-1} L_i $$
    where each octave $L_i$ is defined as:
    $$ L_i = \sigma_s(p, \omega_i) (\gamma \cdot \text{Albedo}(p, \omega_i))^i L_{light}(\omega_i) P'_i(\omega_i, \omega_o) \exp \left( -a_i \int_0^t \sigma_t(x(s), \omega_i) ds \right) $$
    *   $L_{light}(\omega_i)$: Incident radiance from the light source.
    *   $\text{Albedo}(p, \omega_i)$: Local RGB albedo at point $p$ for direction $\omega_i$. Using RGB albedo (rather than a scalar) preserves spectral energy redistribution [S1].
    *   $P'_i(\omega_i, \omega_o)$: Modified phase function for octave $i$.
    *   $T_i$: Transmittance for octave $i$, using parameter $a$ to control attenuation:
        $$ T_i = \exp \left( -a_i \int_0^t \sigma_t(x(s), \omega_i) ds \right) $$
    *   **Modified Phase Function:** Linearly interpolate between the original hair phase function $P$ and an isotropic phase function $P_{iso}$:
        $$ P'_i(\omega_i, \omega_o) = \text{lerp}(P(\omega_i, \omega_o), P_{iso}, 1 - c^i) $$
        Here, $c \in [0, 1]$ controls the rate of isotropization [S1].
    *   **Evaluation:** The summation is evaluated stochastically by sampling a single octave $i$ according to its relative contribution, or deterministically by accumulating all terms [S1].

7.  **Compositing:**
    Combine the direct geometric contribution and the approximated multiple scattering contribution to produce the final pixel color [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def render_hair_fur(camera_ray, hair_geometry, light_sources, medium_params):
    # 1. First Geometric Interaction
    hit = hair_geometry.intersect(camera_ray)
    if hit is None:
        return [0.0, 0.0, 0.0]  # background_color

    # Compute direct illumination on explicit geometry using next-event estimation
    direct_radiance = [0.0, 0.0, 0.0]
    for light in light_sources:
        shadow_ray = hit.point + light.direction * light.distance
        # Standard next-event estimation with hair BCSDF
        direct_radiance += hit.bcsdf.evaluate(hit.normal, shadow_ray.direction, light.direction) * light.intensity * math.exp(-compute_optical_depth_along_ray(shadow_ray.origin, shadow_ray.direction, medium_params.sigma_t))

    # 2. Transition to Medium
    # Sample outgoing direction from BCSDF at the first hit
    outgoing_dir = hit.bcsdf.sample_outgoing(camera_ray.direction)

    # 3. Single Scattering in Equivalent Medium
    # Sample scattering point p along the view ray using delta tracking
    t_sample = random.uniform(0, camera_ray.length)
    p = camera_ray.point_at(t_sample)

    if p is not None:
        # Select a random light direction
        light_sample = random.choice(light_sources)

        # March along light_sample to evaluate transmittance and direct illumination
        transmittance = math.exp(-compute_optical_depth_along_ray(p, light_sample.direction, medium_params.sigma_t))
        incident_radiance = light_sample.radiance

        # Base single scattering term
        sigma_s = medium_params.sigma_s(p, light_sample.direction)
        albedo = medium_params.albedo(p, light_sample.direction)  # RGB albedo
        phase = medium_params.phase(p, light_sample.direction, outgoing_dir)

        single_scattering_estimate = sigma_s * albedo * incident_radiance * phase * transmittance

        # 4. Multiple Scattering Approximation
        # Stochastic evaluation: sample one octave according to relative contribution
        octave_index = random.choices(range(medium_params.N), weights=compute_octave_weights(medium_params.N, single_scattering_estimate))[0]

        # Calculate octave-specific parameters
        # Gamma moderates saturation
        albedo_term = (medium_params.gamma * albedo) ** octave_index

        # Modified phase function: lerp between original and isotropic
        # c controls isotropization rate
        isotropic_phase = 1.0 / (4.0 * math.pi)
        original_phase = medium_params.phase(p, light_sample.direction, outgoing_dir)
        modified_phase = original_phase + (1.0 - medium_params.c ** octave_index) * (isotropic_phase - original_phase)

        # Attenuation parameter 'a' allows deeper penetration for higher orders
        a_i = medium_params.a * (octave_index + 1)
        octave_transmittance = math.exp(-a_i * compute_optical_depth_along_ray(p, light_sample.direction, medium_params.sigma_t))

        multiple_scattering_contribution = sigma_s * albedo_term * incident_radiance * modified_phase * octave_transmittance

        # Composite
        final_radiance = direct_radiance + multiple_scattering_contribution
    else:
        final_radiance = direct_radiance

    return final_radiance

def compute_optical_depth_along_ray(origin, direction, sigma_t):
    # Standard ray marching integration of sigma_t along the ray
    return 1.0  # [G] Implementation detail: typically computed via adaptive ray marching

def compute_octave_weights(N, base_contribution):
    # Compute relative contributions for stochastic octave sampling
    weights = []
    for i in range(N):
        weights.append(base_contribution * (0.5 ** i))  # [G] Heuristic weighting for relative contribution
    return weights
```

### Edge Cases and Pitfalls
*   **Spectral Variation:** The derived phase function is spectrally varying [S1]. However, the method avoids per-channel sampling by evaluating the phase function only for a deterministic outgoing direction during single-scattering estimation [S1].
*   **Color Saturation:** Using a scalar contribution factor instead of local RGB albedo leads to severe color loss [S1]. Repeated application of albedo can cause excessive saturation, mitigated by the gain parameter $\gamma$ [S1].
*   **Tightly Coiled Hair:** The method struggles to reproduce localized bright spots from intense multiple backscattering in highly scattering, tightly coiled hair (e.g., white Afro), approximating them as macroscopic energy distribution instead [S1].

## Operating characteristics
*   **Performance:** Achieves a 7.1× to 11.06× speedup compared to ground truth path tracing at the same noise level [S1]. It is 2.7× to 3.26× faster than Extended Dual Scattering [S1].
*   **Timing:** On human hair, renders an image in 1.17 minutes on average, compared to 9.98 minutes for path tracing (8.53× speedup) and 2.27 minutes for Dual Scattering (1.94× speedup) [S1]. On fur scenes, achieves an average speedup of 3.84× (1.17 minutes vs. 4.49 minutes) compared to Extended Dual Scattering [S1].
*   **Samples:** Requires 256 SPP for equal visual quality, whereas path tracing and Dual Scattering require 512 SPP [S1].
*   **Strengths:** Preserves high-frequency details via explicit geometry while efficiently capturing soft volumetric glow [S1]. Robust under various lighting conditions (studio, environment, directional, spot) [S1].
*   **Limitations:** Cannot reproduce localized bright spots from intense multiple backscattering in tightly coiled hair [S1]. Assumes dense hair/fur behaves as a continuous medium, which may fail for sparse or lifted flyaway strands where voxel-based accumulation underestimates scattering [S1].

## Background
The method was proposed by Hu, Zhu, Lin, and Zhang, building on the observation that dense hair aggregates behave like an anisotropic participating medium [S1]. It addresses the limitations of prior approximations like Dual Scattering and Fur BSSRDF, which often yield dry or blurred appearances in dense hair [S1]. The implementation was demonstrated in Mitsuba 0.6 [S1]. Parameters were obtained via numerical optimization using Ceres Solver with an L1 loss on a scene with three area lights [S1].

## Sources
[S1] Efficient Fur and Hair Multiple Scattering Using Volumetric Approximation, https://doi.org/10.1145/3811343, fetched 2026-11-01
