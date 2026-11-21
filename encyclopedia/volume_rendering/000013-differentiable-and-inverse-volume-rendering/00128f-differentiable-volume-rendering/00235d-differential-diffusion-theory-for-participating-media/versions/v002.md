# Differential Diffusion Theory for Participating Media

Also called: Differential Diffusion Approximation, Differential Diffusion Equation, Differential Diffusion Method, Differential Diffusion Theory, Diffusion Approximation Derivative, Diffusion-based Differentiable Rendering

## Overview
Differential Diffusion Theory is a differentiable rendering framework for participating media that computes arbitrary scene parameter derivatives (density, absorption/scattering coefficients, anisotropic parameters) by solving the differential form of the diffusion equation [S1]. It replaces expensive Monte Carlo path tracing for multiply-scattered light with a partial differential equation (PDE) solver, enabling real-time optimization of millions of variables [S1]. Unlike standard ray marching which treats scattering as a local process, it decouples the global scattering solution (fluence) from local ray integration, allowing the use of fast PDE solvers instead of heavy precomputation or path tracing [S1].

## How it works
**Inputs:**
*   Scene parameters $\pi$ (density $\rho$, scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, anisotropy $\bar{\mu}$, lighting parameters).
*   Initial radiance fields $L_d^0$ (scalar fluence) and $\vec{L}_d^1$ (vector flux) from a previous iteration or initialization [S1].
*   Target image (for optimization tasks) [S1].

**Outputs:**
*   Rendered radiance $L_{out}$ and its derivative $\partial_\pi L_{out}$ with respect to scene parameters $\pi$ [S1].
*   Updated radiance fields $L_d^0, \vec{L}_d^1$ and their derivatives $\partial_\pi L_d^0, \partial_\pi \vec{L}_d^1$ stored as voxels [S1].

**Data Structures:**
*   **Voxel Grid:** Uniform 3D grid indexed by integer coordinates $(i, j, k)$ with edge length $h$ [S1].
*   **Voxel Content:** Each voxel stores:
    *   $L_d^0$: Scalar average radiance (fluence) [S1].
    *   $\vec{L}_d^1$: Vector first-order moment of radiance (flux) [S1].
    *   $\partial_\pi L_d^0$: Derivative of scalar radiance w.r.t. parameters $\pi$ [S1].
    *   $\partial_\pi \vec{L}_d^1$: Derivative of vector radiance w.r.t. parameters $\pi$ [S1] (computed as $\Sigma_{ijk}^9$ in discrete form) [S1].
    *   Optical coefficients: $\rho, \sigma_s, \sigma_a, \bar{\mu}$ [S1].

**Algorithm Steps:**

1.  **Pre-computation of Optical Derivatives:**
    Compute the derivatives of the transport cross-section $\sigma_{tr}$ and the diffusion coefficient $\kappa$ with respect to $\pi$ [S1].
    $$ \partial_\pi \sigma_{tr} = \partial_\pi \sigma_s \left(1 - \frac{\bar{\mu}}{3}\right) - \frac{\partial_\pi \bar{\mu} \sigma_s}{3} + \partial_\pi \sigma_a \quad (13) $$
    $$ \partial_\pi \kappa(x_u) = -\frac{\partial_\pi \rho(x_u)}{\sigma_{tr} \rho^2(x_u)} - \frac{\partial_\pi \sigma_{tr}}{\sigma_{tr}^2 \rho(x_u)} \quad (12) $$
    Note: $\kappa = \frac{1}{3\sigma_{tr}\rho}$ is the diffusion coefficient [S1].

2.  **Compute Source Term Derivatives:**
    Calculate the derivative of the source term $S(x_u)$ required for the differential diffusion equation [S1].
    $$ \partial_\pi S(x_u) = \partial_\pi \kappa_S(x_u) Q_{ri}^0(x_u) + \kappa_S(x_u) \partial_\pi Q_{ri}^0(x_u) - \frac{\sigma_S}{\sigma_{tr}} \nabla \cdot \partial_\pi \vec{Q}_{ri}^1(x_u) - \frac{\partial_\pi \sigma_S \sigma_{tr} - \partial_\pi \sigma_{tr} \sigma_S}{\sigma_{tr}^2} \nabla \cdot \vec{Q}_{ri}^1(x_u) \quad (14) $$
    Where:
    $$ \partial_\pi Q_{ri}^0(x_u) = \frac{1}{4\pi} \int_{\mathbb{S}^2} \partial_\pi L_{ri}(x_u, \omega') d\omega' \quad (15) $$
    $$ \partial_\pi \vec{Q}_{ri}^1(x_u) = \frac{\partial_\pi \bar{\mu}}{4\pi} \int_{\mathbb{S}^2} L_{ri}(x_u, \omega') \cdot \omega' \, d\omega' + \frac{\bar{\mu}}{4\pi} \int_{\mathbb{S}^2} \partial_\pi L_{ri}(x_u, \omega') \cdot \omega' \, d\omega' \quad (16) $$
    The method assumes no surface models, neglecting the derivative of the interface integral and assuming $L_{ri}$ is continuous on $\mathbb{S}^2$ [S1]. $\partial_\pi L_{ri}$ is derived from boundary conditions (see Appendix B of [S1]).

3.  **Solve Differential Diffusion Equation for $\partial_\pi L_d^0$:**
    Solve the following PDE for $\partial_\pi L_d^0(x_t)$ using an iterative numerical solver (e.g., Jacobi or Gauss-Seidel) [S1]:
    $$ \begin{aligned} & \nabla \partial \kappa(x_t) \cdot \nabla L_d^0(x_t) + \nabla \kappa(x_t) \cdot \nabla \partial L_d^0(x_t) \\ & + \partial_{\kappa} \kappa(x_t) \nabla^2 L_d^0(x_t) + \kappa(x_t) \nabla^2 \partial_{\kappa} L_d^0(x_t) \\ & - \partial_{\kappa} \kappa_d(x_t) L_d^0(x_t) - \kappa_d(x_t) \partial_{\kappa} L_d^0(x_t) + \partial_{\kappa} S(x_t) = 0. \end{aligned} \quad (11) $$
    *   **Initialization:** Use the results from the previous iteration ($L_d^0$ and $\partial_\pi L_d^0$) as initial values to reduce convergence time [S1].
    *   **Iteration Count:** Typically 500 iterations are used for convergence in reported experiments [S1].

4.  **Compute $\partial_\pi \vec{L}_d^1$:**
    Once $\partial_\pi L_d^0$ is obtained, compute the derivative of the first-order moment using:
    $$ \begin{aligned} \partial_\pi \vec{L}_d^1(x_u) &= \partial_\pi \kappa(x_u) (-\nabla L_d^0(x_u) + \kappa_s(x_u) \vec{Q}_{ri}^1(x_u)) \\ &\quad + \kappa(x_u) (-\nabla \partial_\pi L_d^0(x_u) + \kappa_s(x_u) \partial_\pi \vec{Q}_{ri}^1(x_u)) \\ &\quad + \partial_\pi \kappa_s(x_u) \vec{Q}_{ri}^1(x_u). \end{aligned} \quad (17) $$
    In discrete form, this is computed as $\Sigma_{ijk}^9$ [S1].

5.  **Compute Derivative of Multiple Scattering Radiance $\partial_\pi L_{ms}$:**
    Use ray marching to integrate the derivatives along the ray [S1].
    $$ \begin{aligned} \partial_{\pi} L_{ms}(x, \omega) &= \int_{D_b}^{D_a} \tau(x_u, x_b) \mathbf{k}_I(x_u) \partial_{\pi} J_{ms}(x_u, \omega) du \\ &\quad + \int_{D_b}^{D_a} (\partial_{\pi} \tau(x_u, x_b) \mathbf{k}_I(x_u) + \tau(x_u, x_b) \partial_{\pi} \mathbf{k}_I(x_u)) J_{ms}(x_u, \omega) du. \end{aligned} \quad (7) $$
    Where $J_{ms}$ is the multiple scattering radiance term approximated by the diffusion solution:
    $$ J_{ms}(x_u, \mathbf{\omega}) = \Omega(L_d^0(x_u) + \frac{\bar{\mu}}{3} \vec{L}_d^1(x_u) \cdot \mathbf{\omega}) $$
    And its derivative:
    $$ \begin{aligned} \partial_{\pi} J_{ms}(x_u, \mathbf{\omega}) &= \partial_{\pi} \Omega(L_d^0(x_u) + \frac{\bar{\mu}}{3} \vec{L}_d^1(x_u) \cdot \mathbf{\omega}) \\ &+ \Omega(\partial_{\pi} L_d^0(x_u) + \frac{\bar{\mu}}{3} \partial_{\pi} \vec{L}_d^1(x_u) \cdot \mathbf{\omega} + \frac{\partial_{\pi} \bar{\mu}}{3} \vec{L}_d^1(x_u) \cdot \mathbf{\omega}). \end{aligned} \quad (8) $$
    With:
    $$ \partial_{\pi}\Omega = \frac{\sigma_t\partial_{\pi}\sigma_s - \sigma_s\partial_{\pi}\sigma_t}{(\sigma_t)^2} \quad (9) $$
    $$ \partial_\pi \bar{\mu} = \frac{3}{2} \int_{-1}^1 \partial_\pi \mu \mu(u) + \mu \partial_\pi p(u) du \quad (10) $$
    $\tau$ is transmittance, $\mathbf{k}_I$ is extinction coefficient. $\partial_\pi \tau$ and $\partial_\pi \mathbf{k}_I$ are derived from optical coefficients [S1].

6.  **Combine with Single Scattering:**
    Add the derivative of single scattering radiance (computed via standard differentiable DVR or path tracing) to $\partial_\pi L_{ms}$ to get the total gradient $\partial_\pi L_{out}$ [S1]. Combining both enhances reconstruction efficiency and accuracy for complex structures [S1].

**Parameters:**
*   **Solver:** Jacobi or Gauss-Seidel iteration [S1].
*   **Iterations:** 500 for convergence in complex scenes [S1].
*   **Grid:** Uniform 3D voxel grid [S1].
*   **Flux-Limited Diffusion:** Optional modulation of diffusion coefficient to handle transparent regions and prevent non-physical fluxes [S1].

**Edge Cases and Pitfalls:**
*   **Vacuum/Thin Regions:** Classical diffusion approximation fails in transparent regions, leading to non-physical radiative fluxes. Flux-limited diffusion is required to modulate the diffusion coefficient in low-density or high-albedo regions [S1].
*   **Boundary Errors:** Cumulative errors in gradient computation occur at the surface of the volume data or in vacuum regions due to ray marching bias and diffusion approximation limitations [S1].
*   **Surface Models:** The method currently omits the Rendering Equation (RE) and assumes no surface models, neglecting interface integral derivatives [S1].

## Subtopics

### [Volume Rendering with Diffusion](5412-volume-rendering-with-diffusion.md)
Standard diffusion approximation for radiance computation without differentiability. It solves the diffusion equation for the fluence field to approximate multiply-scattered light, decoupling global scattering from local ray integration. Choose this when differentiability is not required and speed is paramount for dense media.

## Operating characteristics
*   **Time Complexity:** Substantially faster than Monte Carlo path tracing for dense media. The method achieves real-time performance on density optimization tasks [S1].
*   **Memory Complexity:** Stores voxel fields for $L_d^0, \vec{L}_d^1$ and their derivatives. Memory usage is proportional to the grid resolution, not the number of samples or pixels [S1].
*   **Measured Performance:**
    *   GPU implementation is ~140x faster than CPU [S1].
    *   512x512 resolution: 300 iterations in 13.3s (22.5 FPS) [S1].
    *   1024x1024 resolution: 300 iterations in 18.2s (16.4 FPS) [S1].
    *   256x256 resolution: 300 iterations in 10.2s (29.4 FPS) [S1].
    *   Complex colored smoke scene (49M parameters): Avg 8s/iteration, 500 iterations for convergence [S1].
    *   Compared to Radiative Backpropagation (RB), it is ~100x faster for density optimization [S1].
*   **Strengths:** Handles millions of parameters efficiently; real-time optimization; avoids high variance of MC methods for multiply-scattered light [S1].
*   **Limitations:** Approximate gradients in thin/vacuum regions; limited to volume-only scenes (no surfaces); requires PDE solver implementation [S1].
*   **When to Use:** Differentiable optimization of dense participating media (smoke, clouds) where multiply-scattered light dominates and surface interactions are negligible or handled separately [S1].
*   **When Not to Use:** Scenes with significant surface interactions, highly transparent media without flux-limiting, or when exact unbiased gradients are required (use MC or RB) [S1].

## Background
Originated from the work of Stam [Sta95] who introduced diffusion theory to computer graphics, and Koerner et al. [KPS*14] who introduced flux-limited diffusion. The specific differentiable framework was proposed by Cen et al. [S1] (2024), published in Computer Graphics Forum (doi: 10.1111/cgf.14956). It is the first differentiable rendering framework to compute scene parameter derivatives based on diffusion approximation [S1]. Implemented in Taichi [HLY*21] for real-time performance [S1].

## Sources
[S1] A Differential Diffusion Theory for Participating Media, https://doi.org/10.1111/cgf.14956, fetched 2026-10-30
