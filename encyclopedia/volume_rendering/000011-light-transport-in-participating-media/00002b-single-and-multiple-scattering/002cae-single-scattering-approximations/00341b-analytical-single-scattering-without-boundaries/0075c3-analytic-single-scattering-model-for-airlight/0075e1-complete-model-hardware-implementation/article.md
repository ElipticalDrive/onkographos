# Complete Model Hardware Implementation

Also called: Combined Airlight and Surface Radiance Shader

## Overview
This method implements the full analytic single scattering model for real-time hardware rendering, combining the surface radiance (direct illumination) and airlight (scattered light) components into a single fragment shader pass [S1]. It solves the problem of efficiently computing the appearance of surfaces immersed in a participating medium with a point light source, addressing the computational cost of precomputed radiance transfer or Monte Carlo integration [S1]. It sets itself apart from sibling methods by integrating both the direct surface reflection and the volumetric scattering contribution into one unified shader, requiring only four texture lookups per pixel to achieve real-time performance on programmable graphics hardware [S1].

## How it works
The method computes the final radiance $L$ at a pixel by summing the attenuated exitant surface radiance $L_p$ and the airlight $L_a$ [S1].

**Inputs and Outputs:**
*   **Inputs:** Geometric data (viewer position, surface point, light source position), medium scattering coefficient $\beta$, light intensity $I_0$, surface material properties (diffuse reflectance $k_d$, specular exponent $n$, specular reflectance $k_s$), and precomputed texture maps for special functions $F$, $G_0$, and $G_n$.
*   **Outputs:** Final fragment color $L$.

**Data Structures:**
*   **Textures:** Three 64x64 floating-point textures are used:
    *   $F$: Stores the integral of the single scattering phase function terms.
    *   $G_0$: Stores the diffuse surface scattering integral term.
    *   $G_n$: Stores the specular surface scattering integral term.
    *   These textures are precomputed once and stored in GPU memory to minimize data transfer [S1].

**Formulas and Parameters:**
The final color is calculated as [S1]:
$$L = e^{-T_{vp}} L_p + L_a \quad (19)$$

Where:
*   $T_{vp}$ is the optical thickness between the viewer and the surface point.
*   $L_p$ is the exitant surface radiance, composed of diffuse ($L_{diffuse}$) and specular ($L_{specular}$) components.
*   $L_a$ is the airlight radiance.

The specific terms are defined as follows [S1]:

1.  **Airlight ($L_a$):**
    $$A_0 = \frac{\beta I_0 e^{-T_{sv} \cos \gamma}}{2\pi D_{sv} \sin \gamma}$$
    $$A_1 = T_{sv} \sin \gamma$$
    $$v = \frac{\pi}{4} + \frac{1}{2} \arctan\left(\frac{T_{vp} - T_{sv} \cos \gamma}{T_{sv} \sin \gamma}\right)$$
    $$L_a = A_0 \left( F(A_1, v) - F(A_1, \gamma/2) \right)$$
    Here, $T_{sv}$ is the optical thickness between the source and viewer, $D_{sv}$ is the distance between source and viewer, and $\gamma$ is the angle between the viewer-source vector and the viewer-surface vector.

2.  **Diffuse Surface Radiance ($L_{diffuse}$):**
    $$d_1 = \frac{k_d e^{-T_{sp}} \cos \theta_s I_0}{D_{sp}^2}$$
    $$d_2 = \frac{k_d I_0 \beta^2}{2\pi T_{sp}} G_0(T_{sp}, \theta_s)$$
    $$L_{diffuse} = d_1 + d_2$$
    Here, $T_{sp}$ is the optical thickness between source and surface, $D_{sp}$ is the distance between source and surface, and $\theta_s$ is the angle of incidence.

3.  **Specular Surface Radiance ($L_{specular}$):**
    $$s_1 = \frac{k_s e^{-T_{sp}} \cos^n \theta_{0s} I_0}{D_{sp}^2}$$
    $$s_2 = \frac{k_s I_0 \beta^2}{2\pi T_{sp}} G_n(T_{sp}, \theta_{0s})$$
    $$L_{specular} = s_1 + s_2$$
    Here, $\theta_{0s}$ is the half-vector angle.

**Pseudocode:**
The following Python-like pseudocode implements the fragment shader logic described in the source [S1]. It assumes standard vector math operations are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_fragment_color(F_tex, G0_tex, Gn_tex, 
                           viewer_pos, surface_pos, light_pos,
                           beta, kd, ks, n, I0):
    # Calculate geometric distances and optical thicknesses
    # D_sv: distance between viewer and light
    # D_sp: distance between surface and light
    # D_vp: distance between viewer and surface
    D_sv = length(light_pos - viewer_pos)
    D_sp = length(light_pos - surface_pos)
    D_vp = length(surface_pos - viewer_pos)
    
    # Optical thicknesses: T = beta * distance
    T_sv = beta * D_sv
    T_sp = beta * D_sp
    T_vp = beta * D_vp
    
    # Angles
    # gamma: angle between viewer-light vector and viewer-surface vector
    # Note: Source implies gamma is derived from geometry. 
    # Vector from viewer to light: V_L = light_pos - viewer_pos
    # Vector from viewer to surface: V_S = surface_pos - viewer_pos
    # gamma = acos(dot(normalize(V_L), normalize(V_S)))
    gamma = acos(dot(normalize(light_pos - viewer_pos), normalize(surface_pos - viewer_pos)))
    
    # theta_s: angle of incidence (between surface normal and light direction)
    # Assuming surface normal is provided or computed from geometry
    # theta_s = acos(dot(normalize(light_pos - surface_pos), surface_normal))
    # [G] Surface normal is required but not explicitly listed as input in notes. 
    # Assuming it is available in the shader context.
    theta_s = acos(dot(normalize(light_pos - surface_pos), surface_normal))
    
    # theta_0s: half-vector angle for specular
    # [G] Computed from half vector between view and light directions
    half_vec = normalize(normalize(viewer_pos - surface_pos) + normalize(light_pos - surface_pos))
    theta_0s = acos(dot(normalize(light_pos - surface_pos), half_vec))
    
    # --- Airlight Calculation ---
    # Avoid division by zero if gamma is 0 or pi
    if sin(gamma) < 1e-6:
        airlight = 0.0
    else:
        A0 = (beta * I0 * exp(-T_sv * cos(gamma))) / (2.0 * pi * D_sv * sin(gamma))
        A1 = T_sv * sin(gamma)
        
        # Calculate v parameter
        numerator = T_vp - T_sv * cos(gamma)
        denominator = T_sv * sin(gamma)
        v = (pi / 4.0) + 0.5 * atan2(numerator, denominator)
        
        # Texture lookups for F
        # texRECT(F, float2(A1, v))
        f1 = F_tex.sample(A1, v)
        # texRECT(F, float2(A1, gamma/2))
        f2 = F_tex.sample(A1, gamma / 2.0)
        
        airlight = A0 * (f1 - f2)

    # --- Diffuse Calculation ---
    # Direct term
    d1 = (kd * exp(-T_sp) * cos(theta_s) * I0) / (D_sp * D_sp)
    
    # Scattering term
    # texRECT(G0, float2(T_sp, theta_s))
    g0_val = G0_tex.sample(T_sp, theta_s)
    d2 = (kd * I0 * beta * beta) / (2.0 * pi * T_sp) * g0_val
    
    diffuse = d1 + d2

    # --- Specular Calculation ---
    # Direct term
    s1 = (ks * exp(-T_sp) * cos(theta_0s)**n * I0) / (D_sp * D_sp)
    
    # Scattering term
    # texRECT(Gn, float2(T_sp, theta_0s))
    gn_val = Gn_tex.sample(T_sp, theta_0s)
    s2 = (ks * I0 * beta * beta) / (2.0 * pi * T_sp) * gn_val
    
    specular = s1 + s2

    # --- Final Color ---
    # L = exp(-T_vp) * (diffuse + specular) + airlight
    # Note: Source formula (19) is L = exp(-T_vp) * L_p + L_a
    # L_p is the exitant radiance (diffuse + specular)
    final_color = airlight + (diffuse + specular) * exp(-T_vp)
    
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** The airlight calculation involves division by $\sin \gamma$ and $D_{sv}$. If the viewer and light are aligned with the surface normal or each other such that $\gamma \approx 0$ or $\pi$, or if distances are zero, numerical instability occurs. The pseudocode includes a guard for $\sin \gamma$ [S1].
*   **Optical Thickness Range:** The model is highly accurate for optically thin media ($T_{sv} \le 2$) with less than 4% relative RMS error. For greater optical thicknesses ($T_{sv} > 2$), the error increases slowly but remains viable for real-time scenarios [S1].
*   **Texture Precision:** The use of 64x64 floating-point textures limits the precision of the precomputed integrals. This is a trade-off for performance [S1].

## Operating characteristics
*   **Time Complexity:** Real-time interactive performance in a single rendering pass. The computational cost is dominated by four texture lookups and basic arithmetic operations per pixel/vertex [S1].
*   **Memory Complexity:** Requires storage for three 64x64 floating-point textures ($F, G_0, G_n$). Since these are precomputed once, memory bandwidth usage is minimized [S1].
*   **Measured Performance:** Implemented on an NVidia GeForce 6800 using Cg. Achieved real-time rates for complex scenes (e.g., 39,999 vertices) with multiple light sources. Rendering time is linear with the number of light sources [S1].
*   **Strengths:** Significantly faster than Monte Carlo simulations (which took ~2 hours per glow) while maintaining visual accuracy for real-time applications like games [S1].
*   **Limitations:** Accuracy degrades slightly for very optically thick media ($T_{sv} > 2$), though it remains a viable approximation. Relies on precomputed textures which may not adapt dynamically to changing medium properties without regeneration [S1].

## Background
The method originates from the paper "A Practical Analytic Single Scattering Model for Real Time Rendering" by Ramamoorthi et al. [S1]. It was developed to address the limitations of previous methods that either ignored scattering or relied on expensive Monte Carlo simulations. The model leverages the analytic solution for single scattering in a homogeneous medium to provide a practical, hardware-accelerated approach for real-time graphics [S1].

## Sources
[S1] A Practical Analytic Single Scattering Model for Real Time Rendering, https://cseweb.ucsd.edu/~ravir/papers/singlescat/scattering.pdf, fetched 2026-10-29
