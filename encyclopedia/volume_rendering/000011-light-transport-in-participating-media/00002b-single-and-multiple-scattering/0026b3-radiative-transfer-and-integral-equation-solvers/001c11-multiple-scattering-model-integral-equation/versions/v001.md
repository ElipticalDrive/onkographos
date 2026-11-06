# Multiple Scattering Model (Integral Equation)

Also called: General integral equation for radiation transport, Radiation transport integral equation

## Overview
This method models light transport in participating media by accounting for multiple scattering events, solving for the directional intensity $I(X, \omega)$ at every point $X$ and direction $\omega$. It differs from single-scattering models by including the contribution of light scattered from all other directions, resulting in a self-referential integral equation where the unknown intensity appears on both sides, necessitating iterative or expansion-based solution techniques [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The geometry of the volume, the extinction coefficient $\tau(X)$, the scattering phase function $r(X, \omega, \omega')$, the nondirectional glow source $e(X)$ (e.g., black body thermal radiation), and the boundary illumination $I_0$.
*   **Output:** The directional intensity $I(X, \omega)$ throughout the volume.

**Data Structures:**
The method operates on continuous fields. In a discrete implementation, $I$ is stored as a tensor or array indexed by spatial coordinates and discrete angular directions. The phase function $r$ requires storage for the angular dependency.

**Formulas:**
The core relationship is the general integral equation for radiation transport [S1, unofficial]:

$$I(X, \omega) = I_0(X + D\omega, \omega) \exp\left(-\int_0^D \tau(X + t\omega) dt\right) + \int_0^D \left[ \int_{4\pi} r(X + s\omega, \omega, \omega') I(X + s\omega, \omega') d\omega' + e(X + s\omega) \right] \exp\left(-\int_0^s \tau(X + t\omega) dt\right) ds$$

Where:
*   $X$: The starting point of the viewing ray.
*   $\omega$: The viewing direction.
*   $D$: The distance to the boundary of the volume in direction $\omega$.
*   $I_0(X + D\omega, \omega)$: The external illumination entering the volume at the boundary point $X + D\omega$ from direction $\omega$.
*   $\tau(X)$: The extinction coefficient at point $X$.
*   $r(X, \omega, \omega')$: The scattering phase function, representing the probability density of scattering from incoming direction $\omega'$ to outgoing direction $\omega$ at point $X$.
*   $I(X + s\omega, \omega')$: The unknown intensity at a point $s$ units along the ray from $X$, in direction $\omega'$.
*   $e(X + s\omega)$: The nondirectional glow (emissive source) at point $X + s\omega$.
*   $s$: The integration variable along the ray, from $0$ to $D$.
*   $t$: The integration variable for optical depth calculation.

**Steps:**
1.  **Discretize the Domain:** Discretize the spatial domain into voxels or cells and the angular domain into discrete directions or basis functions.
2.  **Initialize:** Initialize the intensity field $I(X, \omega)$, often with zero or a single-scattering estimate.
3.  **Iterate:** Because $I(X, \omega)$ appears on both sides of the equation, use an iterative solver (e.g., Jacobi, Gauss-Seidel) or an expansion method (e.g., P-N method, Discrete Ordinates) to solve for $I$.
    *   In each iteration, compute the source term $g(s)$ for each point and direction:
        $$g(s) = \int_{4\pi} r(X + s\omega, \omega, \omega') I_{old}(X + s\omega, \omega') d\omega' + e(X + s\omega)$$
    *   Update the intensity using the integral equation, treating the source term as known from the previous iteration.
4.  **Converge:** Repeat until the change in $I(X, \omega)$ between iterations is below a threshold.

**Pitfalls:**
*   **Self-Reference:** The unknown $I(X, \omega)$ appears on both sides, making direct solution impossible without iteration or approximation [S1, unofficial].
*   **Angular Discretization:** Approximating the integral over $4\pi$ requires careful choice of angular quadrature or basis functions to avoid artifacts.
*   **Computational Cost:** The nested integrals (spatial and angular) are computationally expensive, especially for dense media with high scattering albedo.

## Subtopics

### [Full Propagator Approximation for Multiply-Scattered Radiance](1298-full-propagator-approximation-for-multiply-scattered-radianc.md)
Approximates the path integral over a reduced subset of paths surrounding the Most Probable Path (MPP).

### [P-N Method (Spherical Harmonics)](1395-p-n-method-spherical-harmonics.md)
Expands the directional intensity into spherical harmonics, converting the integral equation into a coupled system of partial differential equations.

### [Monte Carlo Method for Multiple Scattering](1940-monte-carlo-method-for-multiple-scattering.md)
Stochastically traces sample rays to estimate the intensity, providing an unbiased but noisy solution.

### [Diffusion Approximation](2372-diffusion-approximation.md)
Replaces the integral equation with a partial differential equation based on the first two angular moments, effective in dense, highly opaque media.

## Operating characteristics
*   **Time Complexity:** High, due to the need to solve a large system of equations or perform many iterations. The complexity depends on the spatial and angular discretization.
*   **Memory Complexity:** High, as it requires storing the intensity field for all spatial points and directions.
*   **Strengths:** Physically accurate for multiple scattering; handles complex geometries and anisotropic scattering.
*   **Limitations:** Computationally expensive; difficult to implement efficiently for real-time applications; sensitive to angular discretization errors.
*   **When to Use:** Offline rendering where accuracy is paramount; scenes with dense, highly scattering media where single scattering is insufficient.
*   **When Not to Use:** Real-time applications; scenes with low scattering albedo where single scattering is a good approximation.

## Background
The general integral equation for radiation transport originates from thermal engineering and neutron transport theory [S1, unofficial]. It was adapted for computer graphics by Kajiya and Von Herzen, who expanded the intensity in spherical harmonics to solve the equation, leading to the P-N method [S1, unofficial]. The equation is fundamental in understanding light transport in participating media, providing a rigorous framework for multiple scattering.

## Sources
[S1] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
