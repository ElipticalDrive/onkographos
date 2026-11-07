# Multiple Scattering Model (Integral Equation)
Also called: General integral equation for radiation transport, Radiation transport integral equation

## Overview
The Multiple Scattering Model (Integral Equation) is a deterministic method for solving the Radiative Transfer Equation (RTE) by explicitly accounting for multiple scattering events in participating media [S1, unofficial]. It formulates the radiance at a point as a path integral that includes in-scattering from all directions along the ray, creating a self-referential equation where the unknown intensity appears on both sides [S1, unofficial]. Unlike single-scattering approximations, this model captures the complex light transport in dense or high-albedo media, distinguishing it from its siblings by providing the exact integral formulation that other methods (like Monte Carlo or P-N) approximate [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Medium properties (extinction coefficient $\tau$, phase function $r$, emission $e$), geometry (ray origin $X$, direction $\omega$, length $D$), and boundary illumination $I_0$.
*   **Output:** Directional intensity $I(X, \omega)$ at position $X$ in direction $\omega$.

**Data Structures**
*   **Ray:** Defined by origin $X$, direction $\omega$, and length $D$.
*   **Medium:** Defined by spatially varying coefficients $\tau(X)$, $r(X, \omega, \omega')$, and $e(X)$.
*   **Integration:** Numerical quadrature over path length $s \in [0, D]$ and solid angle $\Omega = 4\pi$.

**Method Steps**
1.  **Define the Source Term:** Calculate the source term $g(s)$ at a point $X+s\omega$ along the ray. This term represents the total radiance added to the ray at that point from in-scattering and emission [S1, unofficial]:
    $$ g(s) = \int_{4\pi} r(X+s\omega, \omega, \omega') I(X+s\omega, \omega') d\omega' + e(X+s\omega) $$
    where $r$ is the phase function, $I$ is the unknown incoming intensity from direction $\omega'$, and $e$ is nondirectional glow (e.g., black body thermal radiation) [S1, unofficial].

2.  **Formulate the Integral Equation:** Substitute $g(s)$ into the general solution for radiance transport. The intensity $I(X, \omega)$ is the sum of attenuated boundary illumination and the accumulated source term along the ray [S1, unofficial]:
    $$ I(X, \omega) = I_0(X + D\omega, \omega) \exp\left(-\int_0^D \tau(X + t\omega) dt\right) + \int_0^D g(s) \exp\left(-\int_0^s \tau(X + t\omega) dt\right) ds $$
    Combining these yields the general integral equation for radiation transport [S1, unofficial]:
    $$ I(X, \omega) = I_0(X + D\omega, \omega) e^{-\tau_{total}} + \int_0^D \left[ \int_{4\pi} r(X+s\omega, \omega, \omega') I(X+s\omega, \omega') d\omega' + e(X+s\omega) \right] e^{-\tau_{partial}(s)} ds $$
    where $\tau_{total}$ is the optical depth from $X$ to the boundary, and $\tau_{partial}(s)$ is the optical depth from $X+s\omega$ to the boundary [S1, unofficial].

3.  **Solve for Intensity:** Since $I(X, \omega)$ appears on both sides of the equation (inside the integral), it cannot be solved directly [S1, unofficial]. Iterative methods (e.g., source iteration) or expansion methods (e.g., spherical harmonics, discrete ordinates) must be used to resolve the dependency [S1, unofficial].

**Parameters**
*   $\tau(X)$: Extinction coefficient (units $m^{-1}$).
*   $r(X, \omega, \omega')$: Phase function (dimensionless).
*   $e(X)$: Emission coefficient (units $W \cdot m^{-3} \cdot sr^{-1}$).
*   $D$: Ray length through the medium.

**Pitfalls**
*   **Circular Dependency:** The unknown $I(X, \omega)$ is inside the integral, requiring iterative solvers which may converge slowly in highly scattering media [S1, unofficial].
*   **Angular Resolution:** The integral over $4\pi$ requires careful discretization (e.g., Gaussian quadrature) to avoid aliasing artifacts in the phase function [S1, unofficial].
*   **Computational Cost:** Evaluating the double integral (path and angle) for every sample is expensive without approximation [S1, unofficial].

## Subtopics

### [Full Propagator Approximation for Multiply-Scattered Radiance](1298-full-propagator-approximation-for-multiply-scattered-radianc.md)
Approximates the path integral by integrating over a Gaussian support perpendicular to the Most Probable Path (MPP), reducing computational cost compared to full path space integration.

### [P-N Method (Spherical Harmonics)](1395-p-n-method-spherical-harmonics.md)
Solves the integral equation by expanding intensity into spherical harmonics, converting the integral equation into a coupled system of partial differential equations.

### [Monte Carlo Method for Multiple Scattering](1940-monte-carlo-method-for-multiple-scattering.md)
Stochastically traces rays to solve the integral equation, providing unbiased results but with inherent noise, unlike deterministic solvers.

### [Diffusion Approximation](2372-diffusion-approximation.md)
Replaces the integral RTE with a PDE based on the first two angular moments (fluence and flux), effective in dense, highly opaque media.

### Neural Network RTE Simplification via Albedo Series
Simplifies the classical RTE by expressing the co-scattering term $F(p, \omega)$ as an infinite series of albedo powers $\eta^i$, enabling neural network prediction [S2, unofficial]. The output radiance $L_o$ is given by $L_o(p, \omega) = T(p, z)L_i(z, \omega) + \int_p^z T(p, x)\mu_s(x)F(x, \omega)dx$, where $T$ is transmittance and $\mu_s$ is the scattering coefficient [S2, unofficial]. The co-scattering term is expanded as $F(p, \omega) = \sum_{i=0}^{\infty} \eta^i F_i(p, \omega)$, with $F_0 = f_p T I$ and $F_{i+1} = \Pi F_i$, where $\Pi$ is an integral operator [S2, unofficial]. This allows the network to infer the full scattering series from approximate terms without complex iterative calculations [S2, unofficial].

## Operating characteristics
*   **Complexity:** The method is computationally intensive due to the nested integrals over path length and solid angle. Solving the resulting system typically requires iterative methods with complexity dependent on the number of scattering events and angular discretization [S1, unofficial].
*   **Accuracy:** Provides the exact formulation for multiple scattering, serving as the ground truth for approximations [S1, unofficial].
*   **Limitations:** Direct implementation is intractable for real-time rendering due to the infinite series of scattering events and angular dependencies [S1, unofficial]. It is primarily used as a theoretical foundation or for offline precomputation [S1, unofficial].
*   **Use Cases:** High-fidelity offline rendering, validation of approximate methods, and scientific simulation in thermal engineering and neutron transport [S1, unofficial].

## Background
The general integral equation for radiation transport originates from thermal engineering and neutron transport physics [S1, unofficial]. It was adapted for computer graphics by Kajiya and Von Herzen, who expanded the intensity $I(X, \omega)$ in spherical harmonics to solve the equation [S1, unofficial]. The formulation is mathematically equivalent to the P-N method used in thermal engineering [S1, unofficial].

## Sources
[S1] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S2] A real-time rendering method for high albedo anisotropic materials with multiple scattering, https://doi.org/10.48550/arxiv.2401.14051, fetched 2026-10-26 (unofficial)
