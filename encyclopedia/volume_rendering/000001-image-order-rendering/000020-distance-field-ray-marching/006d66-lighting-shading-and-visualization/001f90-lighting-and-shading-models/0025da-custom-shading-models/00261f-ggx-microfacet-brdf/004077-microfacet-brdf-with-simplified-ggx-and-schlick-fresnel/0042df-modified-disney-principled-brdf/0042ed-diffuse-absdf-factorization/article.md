# Diffuse ABSDF Factorization

Also called: Diffuse Aggregation

## Overview
Diffuse ABSDF Factorization approximates the diffuse component of light transport in participating media by separating the integration over material parameters from the integration over geometry and orientation [S1, unofficial]. It reduces the double integral into a product of a parameter expectation term and a geometric convolution term [S1, unofficial]. It sets itself apart from sibling methods like [Specular ABSDF Factorization](10872-specular-absdf-factorization.md) by focusing specifically on the diffuse lobe, which allows for the use of Spherical Gaussian (SG) approximations for the cosine-weighted normal distribution [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Joint probability density function (PDF) of material parameters $p(\beta_c, \beta_m)$, shading normal distribution $p_N(n)$, incoming direction $\omega_i$, and outgoing direction $\omega_o$ [S1, unofficial].
*   **Outputs:** The aggregated diffuse scattering coefficient or integral value [S1, unofficial].

**Data Structures:**
*   **Material Parameters:** $\beta_c, \beta_m \in [0, 1]$ [S1, unofficial].
*   **Normal Distribution:** $p_N(n)$ defined on the hemisphere $S^2$ [S1, unofficial].
*   **Spherical Gaussian (SG):** Defined by amplitude $c$, concentration $\kappa$, and axis $\omega_h$ [S1, unofficial].

**Steps:**
1.  **Split the Integral:** Decompose the joint PDF $p_Y(\gamma)$ into a product of marginal PDFs for material parameters and normals: $p_Y(\gamma) \approx p(\beta_c, \beta_m) p_N(n)$ [S1, unofficial].
2.  **Evaluate Parameter Integral:** Compute the expectation term for the material parameters over the unit square $[0,1]^2$:
    $$ \int_{[0,1]^2} (1 - \beta_m) \beta_c p(\beta_c, \beta_m) d\beta_c d\beta_m = E[\beta_c] - E[\beta_m \beta_c] $$
    [S1, unofficial].
3.  **Approximate Geometric Integral:** Approximate the clamped dot product function $\langle n \cdot \omega_i \rangle \langle n \cdot \omega_o \rangle$ using a Spherical Gaussian (SG) product:
    $$ \langle n \cdot \omega_i \rangle \langle n \cdot \omega_o \rangle \approx c \cdot SG(\omega_h; n, \kappa) $$
    where $c$ is the amplitude and $\kappa$ is the concentration of the resulting SG, and $\omega_h$ is the half-vector between $\omega_i$ and $\omega_o$ [S1, unofficial].
4.  **Convolve with Normal Distribution:** The remaining integral becomes a convolution between the approximated SG and the normal distribution $p_N(n)$:
    $$ \int_{S^2} c \cdot SG(\omega_h; n, \kappa) p_N(n) dn $$
    [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def diffuse_absdf_factorization(p_beta_c, p_beta_m, p_N, omega_i, omega_o):
    # 1. Split joint PDF into marginal PDFs for parameters and normals
    # p_Y(gamma) approx p(beta_c, beta_m) * p_N(n)

    # 2. Evaluate parameter integral: E[beta_c] - E[beta_m * beta_c]
    E_beta_c = integrate(p_beta_c, lambda b: b, bounds=[0, 1])
    E_beta_m_beta_c = integrate(p_beta_c, p_beta_m, lambda bc, bm: bm * bc, bounds=[0, 1], [0, 1])
    param_term = E_beta_c - E_beta_m_beta_c

    # 3. Approximate geometric integral using Spherical Gaussian (SG)
    # <n . omega_i> <n . omega_o> approx c * SG(omega_h; n, kappa)
    omega_h = normalize(omega_i + omega_o)
    # [G] Standard SG fitting procedure for clamped dot products
    c, kappa = fit_sg_for_clamped_cosines(omega_i, omega_o) 

    # 4. Convolve SG with normal distribution p_N(n)
    geo_term = integrate(p_N, lambda n: c * sg(omega_h, n, kappa))

    return param_term * geo_term
```

**Parameters:**
*   **SG Approximation:** The specific values for $c$ and $\kappa$ depend on the angle between $\omega_i$ and $\omega_o$ [S1, unofficial]. Exact formulas are provided in Wang et al. [2009] [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Orientation-Varying Parameters:** The basic factorization assumes orientation-independent parameters. If parameters vary with orientation, the splitting assumption breaks down and requires handling as described in §5.4 of [S1, unofficial].
*   **SG Axis Dependency:** The product SG becomes a function of the half vector $\omega_h$ [S1, unofficial].
*   **SG Approximation Error:** The SG approximation of the clamped dot product introduces error, particularly at grazing angles or high roughness [S1, unofficial].

## Subtopics

### Cosine Term Calculation for Asymmetric BSDF
This variant calculates the correct cosine term for the path space measurement equation when the BSDF is not symmetric with respect to the geometric normal, such as in photon tracing [S2, unofficial]. It prevents light leaks by checking the alignment of directions with normals [S2, unofficial].
*   Calculate dot products: $wi\_ns = wi \cdot n_s$, $wi\_ng = wi \cdot n_g$, $wo\_ns = wo \cdot n_s$, $wo\_ng = wo \cdot n_g$ [S2, unofficial].
*   If $wi\_ng \cdot wi\_ns \le 0$ or $wo\_ng \cdot wo\_ns \le 0$, return 0 [S2, unofficial].
*   If transport direction is FROM_CAMERA, return $|wi\_ns|$ [S2, unofficial].
*   If transport direction is FROM_LIGHT, return $\frac{|n_s \cdot \omega_o| \cdot |n_g \cdot \omega_i|}{|n_g \cdot \omega_o|}$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** The method reduces the evaluation of diffuse scattering to moment calculations and a closed-form SG convolution [S1, unofficial].
*   **Limitations:** Approximation error is introduced by the SG fitting of the clamped dot product [S1, unofficial].
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
Diffuse ABSDF Factorization originates from the work on Efficient Scene Appearance Aggregation for Level-of-Detail Rendering [S1, unofficial]. It builds on the concept of aggregating material properties to simplify light transport calculations [S1, unofficial]. The use of Spherical Gaussians for approximating BRDF terms is based on the work of Wang et al. [2009] [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
[S2] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
