# Forward Scattering Dipole Model

Also called: Effective BRDF, Forward scattering dipole, Functional Integral Approximation for Radiative Transport, Functional Integral Dipole Model, Polymer Physics Dipole Model

## Overview
The Forward Scattering Dipole Model is an analytical subsurface scattering method derived from the functional integral formulation of the Radiative Transfer Equation (RTE) under the explicit assumption of strong forward scattering [S1]. It solves the problem of accurately modeling light transport in optically dense, highly anisotropic media (such as skin, fog, or snow) where standard diffusion approximations fail due to their isotropic scattering assumptions [S1]. Unlike isotropic diffusion models [Jensen et al. 2001, d'Eon 2014] or the directional dipole model [Frisvad et al. 2014], this method explicitly encodes the anisotropy of the phase function through a single scaling parameter, capturing the full behavior from unscattered straight-line propagation to the fully diffusive limit without requiring separate Monte Carlo solutions for single-scattered contributions [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Medium volume $V$, scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, extinction coefficient $\sigma_t = \sigma_s + \sigma_a$, reduced scattering coefficient $\sigma'_s$, phase function anisotropy parameter $g$ (or width $\mu$), boundary BSDF $f_s$, and surface normal $\hat{n}$.
*   **Outputs:** Incident radiance $L_i$ at a point $\vec{x}$ on the interior surface in direction $-\hat{\omega}$, or the effective forward scattering BRDF $f_r$.

**Data Structures:**
The core data structure is the propagator $G$, which represents the Green's function for the RTE. In the forward scattering approximation, $G$ is defined via a functional integral over path tangent vectors $\hat{\beta}(s)$ [S1].

**Parameters:**
*   $\mu$: Phase function width parameter. For heavily forward peaked scattering ($\mu \ll 1$), the mean cosine $g \approx 1 - \mu$ and $\sigma'_s = \mu \sigma_s$ [S1]. Typical values for skin: $g \in [0.7, 0.8]$ [S1].
*   $n$: Index of refraction (e.g., $n=1.4$ for skin) [S1].
*   $\rho$: Beckmann microfacet roughness (e.g., $\rho=0.35$) [S1].

**Method Steps:**

1.  **Define the Phase Function:**
    Use a forward-peaked Gaussian phase function $P_{\text{gauss}}$:
    $$ P_{\text{gauss}}(\hat{\omega}, \hat{\omega}') = N \exp\left(-\frac{1 - \hat{\omega} \cdot \hat{\omega}'}{\mu}\right) $$
    where $N$ is the normalization constant ensuring $\int P d^2\hat{\omega}' = 1$ [S1].

2.  **Construct the Propagator $G$:**
    The propagator $G(s, \vec{x}, \hat{\omega}, \vec{x}', \hat{\omega}')$ is the solution to the RTE with a Dirac delta source and **fully absorbing boundary conditions** (i.e., $G=0$ for outgoing directions at the boundary $\partial V$) [S1]. It is approximated by the functional integral:
    $$ G(s, \vec{x}, \hat{\omega}, \vec{x}', \hat{\omega}') = N_G e^{-\sigma_a s} \int [D\hat{\beta}] \delta^2[\hat{\beta}(0) - \hat{\omega}'] \delta^2[\hat{\beta}(s) - \hat{\omega}] \delta^3\left[\vec{x} - \vec{x}' - \int_0^s \hat{\beta}(s') ds'\right] \exp \left[ -\frac{1}{2\sigma'_s} \int_0^s \left| \frac{d\hat{\beta}(s')}{ds'} \right|^2 ds' \right] $$
    Here, $\hat{\beta}(s')$ is the unit-length tangent vector at path length $s'$. The exponential term represents a quadratic bending energy [S1].
    *Note:* The exact evaluation of this functional integral is difficult because $\hat{\beta}$ must remain a unit vector. The model relies on the approximation that ignores or handles this constraint to yield a Gaussian functional [S1].

3.  **Evaluate the Rendering Equation:**
    The incident radiance $L_i(s, \vec{x}, \hat{\omega})$ on the interior surface is computed by coupling $G$ with the boundary BSDF $f_s$:
    $$ L_i(s, \vec{x}, \hat{\omega}) = \int ds' dA(\vec{x}') d^2\hat{\omega}' G(s-s', \vec{x}, -\hat{\omega}, \vec{x}', \hat{\omega}') |\hat{\omega}' \cdot \hat{n}(\vec{x}')| \left[ L_e(s', \vec{x}', \hat{\omega}') + \int d^2\hat{\omega}'' L_i(s', \vec{x}', \hat{\omega}'') f_s(\vec{x}', \hat{\omega}'' \leftrightarrow \hat{\omega}') |\hat{\omega}'' \cdot \hat{n}(\vec{x}')| \right] $$
    This equation is solved recursively [S1].

4.  **Recursive Sampling (Monte Carlo Integration):**
    To evaluate the rendering equation, use an importance sampling scheme [S1]:
    *   **Step 1:** Set the initial intersection point $\vec{x}_i = \vec{x}_o$ (the point where light enters the medium).
    *   **Step 2:** Sample the index-matched forward scattering BRDF $f_r$ (derived from $G$) and the medium boundary BSDF $f_s$.
    *   **Step 3:** Repeat sampling $f_r$ and $f_s$ recursively.
    *   **Termination:** Stop when $f_s$ generates a query direction pointing out of the medium.
    *   **Weighting:** For subsequent steps after the first, calculate weights based on the planar propagator $G^p_f$ instead of the point propagator $G_b$ [S1].

5.  **Effective BRDF Approximation:**
    For index-matched media, the effective forward scattering BRDF is given by:
    $$ f_r(\hat{\omega}_{in} \leftrightarrow \hat{\omega}_{out}) = \int G_{pb}(s, \hat{\omega}_{out}, -\hat{\omega}_{in}) ds $$
    where $G_{pb}$ is the planar propagator [S1]. This BRDF can be sampled in constant time and serves as a Level of Detail (LOD) approximation for optically dense media [S1].

**Pitfalls:**
*   **Unit Vector Constraint:** The functional integral requires $\hat{\beta}(s_0)$ to be a unit vector. Ignoring this constraint simplifies the integral to a Gaussian form but introduces approximation errors [S1].
*   **Boundary Conditions:** The propagator $G$ assumes fully absorbing boundaries. Surface interactions (Fresnel, roughness) must be handled externally via the BSDF $f_s$ in the rendering equation; they cannot be embedded in $G$ [S1].
*   **Singularity:** The model has a single singularity corresponding to initial unscattered propagation, which must be handled by the analytical importance sampling scheme [S1].

## Subtopics

### [Infinite Volume Forward Scattering Propagator](10181-infinite-volume-forward-scattering-propagator.md)
The Infinite Volume Forward Scattering Propagator ($G_f$) is an analytical approximation of the radiative transfer propagator for participating media that exhibit strong forward scattering. It solves the problem of accurately modeling light transport in optically dense media where standard diffusion approximations fail due to the directional nature of scattering. Unlike the Bounded Forward Scattering Propagator, which handles finite boundaries, $G_f$ assumes an infinite medium, serving as the foundational kernel for subsurface scattering calculations before boundary corrections are applied. It is distinguished from isotropic diffusion models by explicitly encoding the anisotropy of the phase function through a single scaling parameter, allowing it to capture the transition from ballistic to diffusive transport.

## Operating characteristics
*   **Time Complexity:** The analytical evaluation of the propagator allows for constant-time sampling of the effective BRDF [S1]. The recursive rendering equation evaluation depends on the number of bounces required for $f_s$ to exit the medium, but avoids the long chains of volumetric scattering interactions typical of Monte Carlo methods [S1].
*   **Memory Complexity:** Low, as it relies on analytical formulas rather than storing large lookup tables or path histories, except for the recursive stack depth [S1].
*   **Performance:** Accurately captures ground truth behavior over a wide range of parameters, including the optically dense limit where isotropic diffusion models fail [S1]. It is computationally efficient compared to full Monte Carlo path tracing for forward scattering media [S1].
*   **Strengths:** Explicitly handles strong forward scattering; captures the transition from ballistic to diffusive transport; no need for separate single-scatter solutions; decouples volume transport from surface interactions [S1].
*   **Limitations:** Approximate due to the handling of the unit vector constraint in the functional integral; assumes a Gaussian phase function; fully absorbing boundary assumption for $G$ requires external handling of surface reflections [S1].
*   **When to use:** Translucent materials with strong forward scattering (skin, fog, clouds, snow, organic substances) [S1].
*   **When not to use:** Media with isotropic scattering or weak anisotropy, where standard diffusion models are sufficient and more accurate [S1].

## Background
The Forward Scattering Dipole Model was introduced by Roald Frederickx and Philip Dutré at KU Leuven in 2017 [S1]. It is derived from the functional integral formulation of radiative transport, originally introduced by Tessendorf [1987, 1991, 2003], and links this formulation to the partition function of a worm-like chain in polymer physics [S1]. This connection allows the adaptation of results from polymer physics to derive a practical rendering algorithm for subsurface scattering [S1]. The model addresses the limitations of previous diffusion-based dipole models (Jensen et al. [2001], d'Eon [2014]) and the directional dipole model (Frisvad et al. [2014]) by explicitly assuming strong forward scattering [S1].

## Sources
[S1] A forward scattering dipole model from a functional integral approximation, https://doi.org/10.1145/3072959.3073681, fetched 2026-10-26
