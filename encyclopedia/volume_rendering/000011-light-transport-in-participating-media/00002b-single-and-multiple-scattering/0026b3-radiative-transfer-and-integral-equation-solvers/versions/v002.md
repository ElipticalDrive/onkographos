# Radiative Transfer and Integral Equation Solvers

Also called: Deterministic Radiative Transfer, Integral Equation Methods for Volume Rendering

## Overview
This family solves the Radiative Transfer Equation (RTE) deterministically by discretizing the angular or spatial domains, avoiding the stochastic noise inherent in path tracing. Unlike [Volumetric Path Tracing](1863-volumetric-path-tracing.md) which samples individual light paths, these methods compute global illumination fields (intensities or source terms) through iterative propagation or linear system solving. They are distinct from [Delta tracking](985-delta-tracking.md) in that they do not rely on rejection sampling for free-flight distances but instead propagate energy through structured grids or discrete direction bins.

## Shared principles
All methods in this family are derived from the conservation of energy for a differential cylinder of light passing through a medium [S1]. They share the following core definitions and mechanisms:

**1. Fundamental Interactions**
Any volumetric effect is composed of three fundamental characteristics [S1]:
*   **Absorption:** The loss of radiant energy along a ray due to energy conversion (e.g., to heat). It is described by the absorption coefficient $\sigma_a$ (units: $m^{-1}$) [S1].
*   **Emission:** The addition of radiant energy where the medium is luminous (e.g., flames). It is described by the emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$, representing radiance emitted per unit length [S1].
*   **Scattering:** The redirection of light rays upon collision with particles. It is split into:
    *   *In-scattering:* Light from other directions reflected into the view ray.
    *   *Out-scattering:* Light from the view ray reflected away.
In most lighting calculations, out-scattering is folded into the **extinction** term because its net effect on the view ray is identical to absorption [S1].

**2. Notation**
*   $\mathbf{p}$: Position of interaction.
*   $\boldsymbol{\omega}$: Ray direction (unit vector).
*   $L$: Radiance quantity.
*   $L_i$: Incoming radiance.
*   $L_o$: Outgoing radiance.
*   $\sigma_t$: Total extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [S1].

**3. Differential Forms**
The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss and gain terms [S1]:
*   **Absorption Model:** $dL = -\sigma_a L_i dt$ [S1].
*   **Emission Model:** $dL = \sigma_e dt$ (where $\sigma_e$ is the emission coefficient) [S1].
*   **General RTE:** Combines extinction (loss) and source terms (gain from emission and in-scattering) [S1].

**4. Transmittance**
The fraction of light surviving transmission through a medium is defined by the Beer-Lambert law [S1]:
$$ T = e^{-\tau} $$
where $\tau$ is the optical depth (integrated extinction) [S1].

**5. Phase Functions**
Scattering directionality is modeled by a phase function $p(\omega_i, \omega_o)$, defined in terms of $a = \cos(\theta)$, where $\theta$ is the angle between the incident direction $\omega_i$ and the scattered direction $\omega_o$ [S1]. The choice of phase function depends on the ratio of particle radius $r$ to wavelength $\lambda$ [S1]:
*   **Rayleigh Scattering:** Used when $r \ll \lambda$ (e.g., smoke, dust, water molecules).
    $$ P_R(a) = \frac{3}{4}(1+a)^2 $$
*   **Mie Scattering:** Used when $r \approx \lambda$ (e.g., fog, water droplets). Common variants include:
    *   Haziness: $P_{HM}(a) = 1 + 9 \left( \frac{1+a}{2} \right)^8$
    *   Murkiness: $P_{MM}(a) = 1 + 50 \left( \frac{1+a}{2} \right)^{32}$
*   **Henyey-Greenstein (HG):** An approximation to Mie scattering, widely used in graphics for its efficiency and asymmetry parameter $g$:
    $$ P_{HG}(a) = \frac{1 - g^2}{(1 + g^2 - 2ga)^{\frac{3}{2}}} $$
*   **Schlick's Approximation:** A computationally cheaper approximation to HG that avoids fractional exponentiation:
    $$ P_S(a) = \frac{1-g^2}{(1-ga)^2} $$

## Subtopics

### [Discrete Ordinates Method](1046-discrete-ordinates-method.md)
The Discrete Ordinates Method (DOM) solves the RTE by approximating the continuous angular dependence of radiance with a finite set of $M$ discrete directions chosen to provide optimal Gaussian quadrature over the solid angle. It differs from the P-N method (which uses spherical harmonics) by working directly with directional intensities, and from ray-tracing methods by discretizing the angular domain into bins to mitigate ray effects. It is primarily used for computing multiple scattering in participating media, such as clouds, where it provides a global illumination solution via iterative flux propagation.

### [Multiple Scattering Model (Integral Equation)](1177-multiple-scattering-model-integral-equation.md)
This method models light transport in participating media by accounting for multiple scattering events, solving for the directional intensity $I(X, \omega)$ at every point $X$ and direction $\omega$. It differs from single-scattering models by including the contribution of light scattered from all other directions, resulting in a self-referential integral equation where the unknown intensity appears on both sides, necessitating iterative or expansion-based solution techniques.

### [Zonal Method](1891-zonal-method.md)
The Zonal Method is a technique for calculating multiple scattering in participating media by assuming isotropic scattering, where the source term $g(X)$ depends only on position $X$. This assumption allows the extension of diffuse radiosity methods, originally developed for interreflecting surfaces, to volumetric domains. It is specifically designed for high-albedo media, such as atmospheric clouds, where single-scattering approximations fail. Unlike general integral equation solvers that handle directional phase functions, the Zonal Method simplifies the problem to solving a system of simultaneous linear equations for isotropic source terms, distinguishing it from siblings like the Discrete Ordinates Method which retain angular dependence.

### [Principal-Ordinates Propagation](4121-principal-ordinates-propagation.md)
Principal-Ordinates Propagation (POP) is a deterministic finite-element method for real-time simulation of anisotropic light transport in heterogeneous participating media. It solves the problem of false scattering and ray effects inherent in previous finite-element methods by dynamically decomposing illumination into directional and point sources and propagating them in independent discrete domains aligned with principal light directions. Unlike methods using general bases like spherical harmonics, POP uses a computationally efficient unimodal basis implicitly aligned with the respective principal ordinate, allowing for virtually arbitrary scattering anisotropy and albedo without precomputation.

### [Sparse Linear System Solver for Wave Propagation](4718-sparse-linear-system-solver-for-wave-propagation.md)
This method solves for forward and backward acoustic wave amplitudes in a layered medium by formulating the coupled wave equations as a sparse linear system $A\mathbf{x} = \mathbf{b}$. It captures multiple internal reflections and transmissions at voxel boundaries defined by acoustic impedance contrasts. Unlike integral equation solvers that approximate scattering over volumes, this approach models wave propagation along straight-line ray paths through discrete interfaces, making it distinct from [Discrete Ordinates Method](1046-discrete-ordinates-method.md) and [Multiple Scattering Model (Integral Equation)](1177-multiple-scattering-model-integral-equation.md).

### [Spherical Harmonics and Spherical Gaussians for Radiance Transfer](5599-spherical-harmonics-and-spherical-gaussians-for-radiance-tra.md)
Spherical Harmonics (SH) and Spherical Gaussians (SG) are approximation methods for radiance transfer used in real-time or long-rendering scenarios to avoid expensive on-the-fly calculations of the light transport equation. They approximate the incident radiance field using Spherical Radial Basis Functions (SRBFs), which are then convolved with the Bidirectional Reflectance Distribution Function (BRDF). SH is preferred for diffuse or rough surfaces, while SG is used for higher-frequency glossy terms or unified frameworks handling all-frequency shadows.

### Jacobi Iterative Method for Diffusion
- **Definition**: An iterative solver using the Jacobi method to compute diffusion radiance $L_{0d}$, its derivative $\partial_\pi L_{0d}$, first-order term $L_{1d}$, and its derivative $\partial_\pi L_{1d}$ by solving the discrete diffusion equation and differential diffusion equation [S1].
- **Steps**:
    1. If $L_{0d}$, $L_{1d}$ and their derivatives are uninitialized, assign random values [S1].
    2. Repeat: compute $L_{new}$ using Equation 25; calculate error = MSE($L_{new}$, $L_{0d}$); update $L_{0d} = L_{new}$; stop when error < error_threshold [S1].
    3. Compute $L_{1d}$ using Equation 5 [S1].
    4. Repeat: compute $\partial_\pi L_{new}$ using Equation 29; calculate error = MSE($\partial_\pi L_{new}$, $\partial_\pi L_{0d}$); update $\partial_\pi L_{0d} = \partial_\pi L_{new}$; stop when error < error_threshold [S1].
    5. Return $L_{0d}$, $L_{1d}$, $\partial_\pi L_{0d}$, $\partial_\pi L_{1d}$ [S1].
- **Formulas**:
    - $\partial_\pi L_{d0} = \frac{1}{h^2(\kappa_a)_{i,j,k} + 6\kappa_{ijk}} \left[ \frac{1}{4}(\Sigma^1_{ijk} + \Sigma^2_{ijk}) + \kappa_{ijk} \Sigma^3_{ijk} - h^2(\kappa_a)_{ijk}(L_{d0})_{ijk} + h^2 \Sigma^6_{ijk} \right]$, where $\Sigma^1, \Sigma^2, \Sigma^6$ are defined in Appendix B [S1].
    - Discrete gradient approximation: $\nabla F \approx (F_{i+1,j,k} - F_{i-1,j,k}) / 2h$ [S1].
    - Discrete Laplacian approximation: $\nabla^2 F \approx (F_{i+1,j,k} + F_{i-1,j,k} - 2F_{i,j,k}) / h^2$ [S1].
- **Parameters**:
    - error_threshold: convergence threshold for the mean-square error (MSE) between iterations [S1].

### Differential Form of RTE
- **Steps**: Compute density derivatives directly using the analytical differential expression instead of automatic differentiation [S2].
- **Formulas**:
    - The derivative of transmittance $\tau$ from $v_{x_0}$ to $v_{x_1}$ is the negative product of $\tau$ and the sum of extinction coefficient derivatives $\dot{\kappa}_t$ times step size $\Delta d$ [S2].
    - The derivative of $J_{ss}$ is calculated using Reynolds transport theorem, assuming continuity of $L_{ri}$ in $S^2$ [S2].
- **Parameters**:
    - $\dot{\kappa}_t$ is the derivative of the extinction coefficient, equal to the extinction cross-section $\sigma_t$ [S2].
    - $\Delta d$ is the length of the sampling step in the voxel grid [S2].

### m > 1 Propagation
- **Definition**: Handling of direction bins where $m > 1$ by maintaining work, edge, and corner arrays only for separations $s$ divisible by $m$, and spreading them for non-divisible $s$ [S3, book].
- **Steps**:
    - For $m=2$ in 2-D, compute weights for cells $a, b, c$ from integrals: integral of phase function and segment length. Assign weight in cell $c$ to cell $b$ to reduce spread, normalize $a$ and $b$ to sum to 1 [S3, book].
    - For 3-D $m>1$, use Monte Carlo integration to determine weights due to complex ray/cell intersection geometry [S3, book].
    - On odd-numbered steps, use special weights to propagate flux [S3, book].

## Background
The Discrete Ordinates Method was introduced by [S3, book] as an efficient approach for multiple anisotropic volume scattering, addressing the computational cost of ray-tracing methods for dense media. The Zonal Method, developed by Rushmeier, extends radiosity to participating media by assuming isotropic scattering, enabling the use of form factors similar to surface rendering [S1][S2]. The Integral Equation formulation is a standard theoretical framework for radiative transfer, often solved via iterative methods like Jacobi or Gauss-Seidel when analytical solutions are intractable [S1]. Principal-Ordinates Propagation was introduced to provide real-time deterministic rendering of anisotropic media, overcoming the false scattering artifacts of earlier finite-difference methods [S1]. The application of sparse linear systems for wave propagation in layered media provides a deterministic alternative to stochastic wave-based rendering [S1]. Spherical Harmonics and Spherical Gaussians are widely used in real-time graphics for approximating radiance transfer to avoid expensive integration [S1]. The differential form of the RTE allows for differentiable rendering, enabling gradient-based optimization of volume parameters [S2].

## Sources
[S1] A Differential Diffusion Theory for Participating Media, https://doi.org/10.1111/cgf.14956, fetched 2026-10-30
[S2] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
[S3] Efficient Light Propagation for Multiple Anisotropic Volume Scattering, https://doi.org/10.1007/978-3-642-87825-1_7, fetched 2026-10-26 (book)
