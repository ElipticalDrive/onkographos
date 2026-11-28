# Analytic Single Scattering Model for Airlight

Also called: Airlight Model, Analytic Single Scattering Airlight Model, Equation 11 Model

## Overview
This method computes the radiance of single-scattered light (airlight) along a viewing ray in a homogeneous participating medium, caused by an isotropic point light source. It solves the problem of real-time variation of viewpoint, lighting, medium properties, and scene geometry by reducing a complex single-scattering integral into an analytic function computation dependent on physical parameters and a lookup of a precomputed 2D smooth function $F(u, v)$ independent of physical parameters [S1]. It sets itself apart from subsurface scattering methods (where sources/viewers are outside the medium) and previous analytic glow models (which often ignore attenuation or lack surface shading integration) by providing a complete real-time system that handles both glows and their effect on surface shading [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scattering coefficient $\beta$ (or extinction coefficient if absorption is present), radiant intensity of the point light $I_0$, distance between source and viewer $D_{sv}$, distance between viewer and closest surface point $D_{vp}$ (or $\infty$), angle between light source and viewing ray $\gamma$.
*   **Outputs:** Single scattered radiance (airlight) $L_a$. The total radiance is $L = L_d + L_a$, where $L_d$ is direct transmission [S1].

**Data Structures:**
*   **Lookup Table for $F(u, v)$:** A 2D texture table representing the special function $F(u, v)$.
    *   **Layout:** 2D array of floating-point values.
    *   **Resolution:** $64 \times 64$ is sufficient for maximum error less than 2% [S1].
    *   **Access:** Accessed via texture mapping (e.g., `texRECT`) in graphics hardware. Bilinear interpolation is preferred for accuracy; nearest neighbor can be used for speed with small loss [S1].
    *   **Precomputation:** Computed once and independent of scene parameters [S1].

**Definitions and Parameters:**
*   $T_{sv} = \beta D_{sv}$: Optical thickness between source and viewer.
*   $T_{vp} = \beta D_{vp}$: Optical thickness between viewer and surface point.
*   $\gamma$: Angle between the light source direction and the viewing ray.
*   $A_0(T_{sv}, \gamma, \beta)$: Normalization parameter.
*   $A_1(T_{sv}, \gamma)$: Single physical parameter in the integrand.
*   $F(u, v)$: Special function defined as $F(u, v) = \int_0^v \exp[-u \tan \xi] d\xi$ [S1].

**Step-by-Step Derivation and Computation:**

1.  **Start with the standard airlight integral:**
    The radiance $L_a$ is given by integrating along the viewing direction $x$ from the viewer to the surface point $D_{vp}$:
    $$ L_a = \int_0^{D_{vp}} \beta k(\alpha) \frac{I_0 e^{-\beta d}}{d^2} e^{-\beta x} dx $$
    where $d$ is the distance from the light source to the integration point, and $k(\alpha)$ is the phase function. Assuming an isotropic phase function normalized to $1/4\pi$:
    $$ L_a = \frac{\beta I_0}{4\pi} \int_0^{D_{vp}} \frac{e^{-\beta \sqrt{D_{sv}^2 + x^2 - 2xD_{sv} \cos \gamma}}}{D_{sv}^2 + x^2 - 2xD_{sv} \cos \gamma} e^{-\beta x} dx $$
    where $d = \sqrt{D_{sv}^2 + x^2 - 2xD_{sv} \cos \gamma}$ by the cosine rule [S1].

2.  **Reduce integral dimensions:**
    Substitute optical thicknesses $T^* = \beta D^*$ and $t = \beta x$. Use the substitution $z = t - T_{sv} \cos \gamma$. The integral becomes:
    $$ L_a = \frac{\beta^2 I_0}{4\pi} e^{-T_{sv} \cos \gamma} \int_{-T_{sv} \cos \gamma}^{T_{vp} - T_{sv} \cos \gamma} \frac{e^{-z - \sqrt{z^2 + T_{sv}^2 \sin^2 \gamma}}}{T_{sv}^2 \sin^2 \gamma + z^2} dz $$
    The integrand now depends on only one physical parameter: $T_{sv} \sin \gamma$ [S1].

3.  **Define auxiliary expressions:**
    $$ A_0(T_{sv}, \gamma, \beta) = \frac{\beta^2 I_0 e^{-T_{sv} \cos \gamma}}{2\pi T_{sv} \sin \gamma} $$
    $$ A_1(T_{sv}, \gamma) = T_{sv} \sin \gamma $$
    The integral simplifies to:
    $$ L_a = A_0 \int_{\gamma/2}^{\frac{\pi}{2} + \frac{1}{2} \arctan \frac{T_{vp} - T_{sv} \cos \gamma}{T_{sv} \sin \gamma}} \exp[-A_1 \tan \xi] d\xi $$
    [S1].

4.  **Evaluate using the special function $F(u, v)$:**
    Define $F(u, v) = \int_0^v \exp[-u \tan \xi] d\xi$. The airlight is computed as:
    $$ L_a = A_0 \left[ F(A_1, v_{max}) - F(A_1, v_{min}) \right] $$
    where:
    $$ v_{max} = \frac{\pi}{4} + \frac{1}{2} \arctan \frac{T_{vp} - T_{sv} \cos \gamma}{T_{sv} \sin \gamma} $$
    $$ v_{min} = \frac{\gamma}{2} $$
    [S1].

5.  **Special Case (No Objects):**
    If $T_{vp} = \infty$ (no objects along the viewing ray):
    $$ L_a = A_0 \left[ F(A_1, \frac{\pi}{2}) - F(A_1, \frac{\gamma}{2}) \right] $$
    Here $A_0$ and $A_1$ depend only on $T_{sv}$ and $\gamma$ [S1].

**Pseudocode:**
The following Python-like pseudocode computes the airlight $L_a$ in a fragment shader context, assuming access to a 2D texture `F` containing the precomputed function $F(u, v)$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_airlight(beta, I0, D_sv, D_vp, gamma, F_texture):
    """
    Computes single scattered radiance (airlight) for a point light in a homogeneous medium.
    
    Args:
        beta: Scattering (or extinction) coefficient.
        I0: Radiant intensity of the point light source.
        D_sv: Distance between source and viewer.
        D_vp: Distance between viewer and closest surface point (use float('inf') if none).
        gamma: Angle between light source and viewing ray (radians).
        F_texture: 2D texture lookup function for F(u, v).
        
    Returns:
        airlight: The computed single scattered radiance L_a.
    """
    # Calculate optical thicknesses
    T_sv = beta * D_sv
    if D_vp == float('inf'):
        T_vp = float('inf')
    else:
        T_vp = beta * D_vp

    # Calculate auxiliary parameters A0 and A1
    # Handle edge case where sin(gamma) is 0 to avoid division by zero
    sin_gamma = sin(gamma)
    if sin_gamma == 0:
        # If gamma is 0, the source is directly in front/behind. 
        # The formula involves division by sin(gamma). 
        # In practice, this limit requires careful handling or specific geometric setup.
        # For the purpose of this implementation based on the source, 
        # we assume generic non-zero gamma or that the caller handles degenerate cases.
        # [G] Guard against division by zero.
        return 0.0 

    A0 = (beta * I0 * exp(-T_sv * cos(gamma))) / (2 * pi * D_sv * sin_gamma)
    A1 = T_sv * sin_gamma

    # Calculate the upper bound for the integral argument v
    # v = pi/4 + 0.5 * arctan((T_vp - T_sv * cos(gamma)) / (T_sv * sin(gamma)))
    if T_vp == float('inf'):
        # Limit as T_vp -> infinity of arctan(...) is pi/2
        v_max = pi / 4 + 0.5 * (pi / 2)
    else:
        arg = (T_vp - T_sv * cos(gamma)) / (T_sv * sin_gamma)
        v_max = pi / 4 + 0.5 * atan(arg)

    # Lower bound
    v_min = gamma / 2

    # Lookup F(A1, v) in the precomputed texture
    # F_texture(u, v) returns F(u, v)
    f1 = F_texture(A1, v_max)
    f2 = F_texture(A1, v_min)

    # Compute airlight
    airlight = A0 * (f1 - f2)
    
    return airlight
```

**Edge Cases and Pitfalls:**
*   **Optical Thickness:** The model is physically accurate (less than 4% relative RMS error compared to Monte Carlo) for optically thin media ($T_{sv} \le 2$). Errors grow slowly for $T_{sv} > 2$ but remain low for $T_{sv} < 10$. For very high optical thicknesses ($T_{sv} > 20$), attenuation dominates and RMS errors decrease [S1].
*   **Table Resolution:** A $64 \times 64$ table for $F(u, v)$ yields a maximum error of less than 2% when averaged over 40,000 parameter values. Bilinear interpolation is recommended [S1].
*   **Previous Methods:** Unlike Max [1986], this model accounts for attenuation. Unlike Biri et al. [2004], it avoids polynomial approximations that cause inaccuracies near the source. Unlike Narasimhan and Nayar [2003], it is valid for optically thin media and handles objects within the medium [S1].

## Operating characteristics
*   **Time Complexity:** Real-time. The integral is reduced to a few arithmetic operations and two texture lookups [S1].
*   **Memory Complexity:** Requires storing a $64 \times 64$ 2D texture for $F(u, v)$, which is small and precomputed once [S1].
*   **Performance:** Runs in real-time, whereas Monte Carlo simulations for similar glows took approximately two hours per glow [S1].
*   **Strengths:** Allows real-time variation of viewpoint, lighting, medium properties, and scene geometry. Handles effects of scattering on surface shading and complex environment map lighting [S1].
*   **Limitations:** Assumes homogeneous media and single scattering. Accuracy decreases as optical thickness increases significantly, though it remains a viable approximation for many real-time scenarios [S1]. Not strictly valid for multiple scattering in optically thick media or near sources in complex scenes [S1].

## Background
The method was proposed by Raviraj Narasimhan and Shree K. Nayar in their paper "A Practical Analytic Single Scattering Model for Real Time Rendering" [S1]. It addresses the limitations of previous analytic glow models (Max [1986], Biri et al. [2004], Narasimhan and Nayar [2003]) which either ignored attenuation, used inaccurate approximations, or could not be easily extended to full real-time rendering systems with surface shading. The derivation builds on the standard airlight integral and uses a hybrid approach of analytic simplification and numerical lookup for the special function $F(u, v)$ [S1].

## Sources
[S1] A Practical Analytic Single Scattering Model for Real Time Rendering, https://cseweb.ucsd.edu/~ravir/papers/singlescat/scattering.pdf, fetched 2026-10-29
