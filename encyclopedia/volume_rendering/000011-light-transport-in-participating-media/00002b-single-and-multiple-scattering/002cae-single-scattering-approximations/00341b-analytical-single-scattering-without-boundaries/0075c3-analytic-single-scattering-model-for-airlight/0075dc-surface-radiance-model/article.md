# Surface Radiance Model

Also called: Surface Radiance Single Scattering Model

## Overview
The Surface Radiance Model computes the outgoing radiance from a surface point immersed in a participating medium by summing the direct transmission from a light source and the single-scattered airlight reflected from that surface. It solves the problem of real-time surface shading in volumetric environments by reducing the complex integral of scattered light into analytic evaluations and 2D texture lookups for Lambertian and Phong BRDFs. Unlike the parent [Analytic Single Scattering Model for Airlight](23327-analytic-single-scattering-model-for-airlight.md), which computes radiance arriving at the viewer from the medium itself, this model computes radiance leaving the surface, which is then attenuated by the viewer-to-surface optical thickness to determine final appearance.

## How it works

### Inputs and Outputs
**Inputs:**
*   $I_0$: Light source intensity.
*   $D_{sp}$: Euclidean distance from light source to surface point $p$.
*   $T_{sp}$: Optical thickness between light source and surface point $p$.
*   $T_{vp}$: Optical thickness between viewer and surface point $p$.
*   $\beta$: Scattering coefficient of the medium.
*   Surface geometry: Normal $\mathbf{n}$, viewing direction $\mathbf{v}$, light direction $\mathbf{l}$.
*   BRDF parameters: Diffuse coefficient $k_d$, specular coefficient $k_s$, Phong exponent $n$.
*   Precomputed textures: $G_0$ (Lambertian), $G_n$ (Phong).

**Outputs:**
*   $L_{final}$: Final radiance at the viewer.

### Data Structures
*   **$G_0$ Texture**: A 2D floating-point texture of size $64 \times 64$. It stores values for the function $G_0(T_{sp}, \theta_s)$.
*   **$G_n$ Texture**: A 2D floating-point texture of size $64 \times 64$. It stores values for the function $G_n(T_{sp}, \theta'_s)$, where $n$ is the Phong exponent.

### Definitions and Formulas

**1. Direct Transmission ($L_{p,d}$)**
The direct light attenuated by the medium:
$$ L_{p,d} = \frac{I_0 e^{-T_{sp}}}{D_{sp}^2} f_r(\theta_s, \phi_s, \theta_v, \phi_v) \cos \theta_s $$
Where:
*   $f_r$ is the BRDF.
*   $\theta_s$ is the angle between the light direction and the surface normal.
*   $\phi_s$ is the azimuthal angle of the light relative to the surface normal.
*   $\theta_v, \phi_v$ are the viewing angles.

**2. Single-Scattered Surface Radiance ($L_{p,a}$)**
The integral of airlight over the hemisphere, weighted by the BRDF:
$$ L_{p,a} = \int_{\Omega_{2\pi}} L_a(\gamma'(\theta_s, \omega_i), T_{sp}, \infty, \beta) f_r(\theta_i, \phi_i, \theta_v, \phi_v) \cos \theta_i d\omega_i $$
Where $L_a$ is the airlight radiance from the parent article, evaluated with viewer distance $\infty$ and optical thickness $T_{sp}$. The angle $\gamma'$ is the angle between the incident direction $\omega_i$ and the source direction.

**3. Special Functions for BRDFs**
To solve the integral analytically, 2D functions are defined.

*   **Lambertian ($f_r = k_d$):**
    The integral reduces to a function $G_0(T_{sp}, \theta_s)$:
    $$ L_{p,a} = k_d \beta^2 \frac{G_0(T_{sp}, \theta_s)}{2\pi T_{sp}} $$
    The total Lambertian radiance is:
    $$ L_P = I_0 k_d \left[ \frac{e^{-T_{sp}}}{D_p^2} \cos \theta_s + \beta^2 \frac{G_0(T_{sp}, \theta_s)}{2\pi T_{sp}} \right] $$
    *Note: The source text [S1] uses $D_p$ in the denominator of the first term in Eq 17. This is likely a typo for $D_{sp}$, but is cited as written.*

*   **Phong ($f_r$ depends on $n$):**
    Using the reparameterization method [Ramamoorthi and Hanrahan 2002], angles are measured from the reflection of the viewing direction. Let $\theta'_s$ be the angle between the source direction and the reflected view direction.
    The function $G_n$ is defined as:
    $$ G_n(T_{sp}, \theta'_s) = \int_{\Omega_{2\pi}} \frac{e^{-T_{sp} \cos \gamma}}{\sin \gamma'} \left[ F(A_1, \frac{\pi}{2}) - F(A_1, \frac{\gamma'}{2}) \right] \cos^n \theta_i d\omega_i $$
    Where $\gamma'$ and $A_1$ are functions of $\theta'_s$ and $\omega_i$.
    The total Phong radiance is:
    $$ L_P = I_0 k_s \left[ \frac{e^{-T_{sp}}}{D_{sp}^2} \cos^n \theta'_s + \beta^2 \frac{G_n(T_{sp}, \theta'_s)}{2\pi T_{sp}} \right] $$

**4. Final Radiance**
The computed surface radiance is attenuated by the viewer-to-surface optical thickness:
$$ L_{final} = (L_{p,d} + L_{p,a}) e^{-T_{vp}} $$

### Parameters
*   **Texture Resolution**: $64 \times 64$ for $G_0$ and $G_n$ [S1].
*   **Range**: The functions are precomputed for $0 \le T_{sp} \le 10$ and $0 \le \theta_s, \theta'_s \le \pi$ [S1].
*   **Phong Exponent $n$**: The texture $G_n$ must be precomputed for the specific $n$ used in the material.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs:
# I0: float, light intensity
# Tsp: float, optical thickness source-to-surface
# Dsp: float, distance source-to-surface
# Tvp: float, optical thickness viewer-to-surface
# beta: float, scattering coefficient
# kd: float, diffuse coefficient
# ks: float, specular coefficient
# n: int, phong exponent
# cos_theta_s: float, cosine of angle between light and normal
# cos_theta_0s: float, cosine of angle between light and reflected view direction
# G0: Texture2D, precomputed Lambertian function
# Gn: Texture2D, precomputed Phong function

def compute_surface_radiance(I0, Tsp, Dsp, Tvp, beta, kd, ks, n, cos_theta_s, cos_theta_0s, G0, Gn):
    # 1. Compute Direct Transmission and Single-Scattered Airlight for Diffuse
    # Term 1: Direct transmission
    # Note: Source [S1] Eq 17 uses D_p^2. We use D_sp^2 as per Eq 14 and context.
    d1 = kd * exp(-Tsp) * cos_theta_s * I0 / (Dsp * Dsp)
    
    # Term 2: Single-scattered airlight
    # Lookup G0(Tsp, theta_s)
    # theta_s is derived from cos_theta_s, or passed as angle if texture coords are angular
    # Assuming texture coords are (Tsp, theta_s) where theta_s is in radians or normalized
    # The source uses float2(Tsp, theta_s). We assume theta_s is the angle.
    theta_s = acos(cos_theta_s)
    g0_val = G0.sample_linear(float2(Tsp, theta_s))
    
    d2 = (kd * I0 * beta * beta) / (2 * PI * Tsp) * g0_val
    
    diffuse_radiance = d1 + d2
    
    # 2. Compute Direct Transmission and Single-Scattered Airlight for Specular
    # Term 1: Direct transmission
    s1 = ks * exp(-Tsp) * pow(cos_theta_0s, n) * I0 / (Dsp * Dsp)
    
    # Term 2: Single-scattered airlight
    # Lookup Gn(Tsp, theta'_s)
    theta_0s = acos(cos_theta_0s)
    gn_val = Gn.sample_linear(float2(Tsp, theta_0s))
    
    s2 = (ks * I0 * beta * beta) / (2 * PI * Tsp) * gn_val
    
    specular_radiance = s1 + s2
    
    # 3. Combine and Attenuate for Viewer
    total_surface_radiance = diffuse_radiance + specular_radiance
    final_radiance = total_surface_radiance * exp(-Tvp)
    
    return final_radiance
```

### Edge Cases and Pitfalls
*   **Division by Zero**: The term $\frac{1}{T_{sp}}$ appears in the airlight component. If $T_{sp} \to 0$, this term diverges. However, $G_0(0, \theta_s)$ also approaches 0. The limit must be handled or the texture precomputation must handle the singularity. The source does not explicitly state the limit value, but notes the functions are smooth [S1]. In implementation, a small epsilon or limit evaluation should be used if $T_{sp} < \epsilon$.
*   **Texture Coordinates**: The texture coordinates for $G_0$ and $G_n$ are $(T_{sp}, \theta)$. Ensure $\theta$ is within the precomputed range $[0, \pi]$ (or $[0, 2]$ as noted in [S1] for the plot range, likely radians). Clamping or wrapping may be necessary if inputs exceed precomputed bounds.
*   **$D_p$ vs $D_{sp}$**: The source text [S1] Eq 17 uses $D_p$ in the denominator for the direct term. Given Eq 14 uses $D_{sp}$, and $D_p$ is not defined elsewhere, it is treated as a typo for $D_{sp}$ in the pseudocode.

## Operating characteristics

*   **Time Complexity**: $O(1)$ per surface point per light source. The computation involves a constant number of arithmetic operations and two 2D texture lookups.
*   **Memory Complexity**: $O(1)$ additional memory for the two $64 \times 64$ floating-point textures ($G_0$ and $G_n$).
*   **Performance**: The method is designed for real-time hardware implementation. On an NVIDIA GeForce 6800, the authors achieved real-time rates for scenes with ~40k vertices and 4 light sources [S1].
*   **Strengths**: Reduces complex integrals to simple lookups; enables real-time rendering of single scattering effects on surfaces; handles varying viewpoint and lighting.
*   **Limitations**: Only models single scattering; assumes isotropic scattering for the airlight component derivation; limited to Lambertian and Phong BRDFs (though the framework could be extended); requires precomputation of textures for different Phong exponents.

## Background

The Surface Radiance Model was introduced by Ramamoorthi and Hanrahan in their 2004 paper "A Practical Analytic Single Scattering Model for Real Time Rendering" [S1]. It extends the analytic single scattering model for airlight to include surface reflections. The method relies on the reparameterization technique for Phong BRDFs introduced by Ramamoorthi and Hanrahan in 2002 [S1]. It is used in real-time graphics to simulate the appearance of objects in fog or smoke.

## Sources

[S1] A Practical Analytic Single Scattering Model for Real Time Rendering, https://cseweb.ucsd.edu/~ravir/papers/singlescat/scattering.pdf, fetched 2026-10-29
