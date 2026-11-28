# Longitudinally Engineered Metasurface for 3D Vectorial Holography

Also called: 3D vectorial holography, longitudinally tunable beam arrays

## Overview
This method achieves full 3D vectorial holography by decomposing a target 3D optical field into an array of quasi-non-diffracting beams, each with independently programmable axial intensity and polarization profiles [S1]. Unlike conventional holographic approaches using spatial light modulators (SLMs) or digital micromirror devices (DMDs), which are often restricted to 2D vectorial holography or lack depth control, this method coordinates the modulation of amplitude, depth, and polarization across an entire volume using longitudinally engineered beam arrays [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Target 3D vectorial optical field defined by axial intensity profile $F_{p;q}(z)$ and polarization state for each beam $(p,q)$ in the array; axial extent $Z_0$; number of Bessel components $N$; free-space wavevector $k_0$; lateral beam positions $\rho_{p;q}$.
*   **Output:** Metasurface unitary Jones matrix profile $\tilde{U}_{meta}(\rho, \phi)$, implemented via nanopillar geometry (dimensions $D_x, D_y$) and orientation $\theta(\rho, \phi)$.

**Data Structures**
*   **Beam Array:** A 2D grid of beams indexed by $(p,q)$.
*   **Longitudinal Response Function:** For each beam $(p,q)$, a complex-valued function $\tilde{F}_{p;q}(z)$ defined over $z \in [0, Z_0]$.
*   **Bessel Coefficients:** Complex coefficients $A_{Lp;q;m}$ and $A_{Rp;q;m}$ for $m \in [-N, N]$.
*   **Metasurface Pixel:** Defined by local orientation $\theta(\rho, \phi)$ and transmission parameters $t_x, t_y, \phi_x, \phi_y$.

**Method Steps**

1.  **Define Longitudinal Response Function:**
    For each beam $(p,q)$, define the longitudinal response function $\tilde{F}_{p;q}(z)$ as the product of a scalar axial intensity profile $F_{p;q}(z)$ and a Jones vectorial function describing the polarization state [S1]:
    $$
    \tilde{F}_{p;q}(z) = F_{p;q}(z) [a_{Lp;q}(z)|L\rangle + a_{Rp;q}(z)|R\rangle]
    $$
    where $|L\rangle$ and $|R\rangle$ are unit vectors for left- and right-circular polarization (LCP/RCP), and $a_{Lp;q}(z)$, $a_{Rp;q}(z)$ are z-dependent complex weighting coefficients satisfying the normalization condition $|a_{Lp;q}(z)|^2 + |a_{Rp;q}(z)|^2 = 1$ [S1].

2.  **Decompose into Spin-Dependent Channels:**
    Decompose $\tilde{F}_{p;q}(z)$ into two spin-dependent complex value functions corresponding to the LCP and RCP channels [S1]:
    $$
    F_{L}^{p;q}(z) = F_{p;q}(z)a_{Lp;q}(z)
    $$
    $$
    F_{R}^{p;q}(z) = F_{p;q}(z)a_{Rp;q}(z)
    $$

3.  **Compute Bessel Coefficients via Fourier-like Synthesis:**
    Calculate the complex weighting coefficients for the $m$-th Bessel beam component in the LCP or RCP channels by projecting the spin-dependent functions onto exponential basis functions [S1]:
    $$
    A_{Lp;q;m} = \frac{1}{Z_0} \int_0^{Z_0} F_{L}^{p;q}(z) e^{-i \frac{2\pi}{Z_0} m z} dz
    $$
    $$
    A_{Rp;q;m} = \frac{1}{Z_0} \int_0^{Z_0} F_{R}^{p;q}(z) e^{-i \frac{2\pi}{Z_0} m z} dz
    $$
    where $Z_0$ is the axial extent, and the discrete spacing is $\Delta k_z = 2\pi/Z_0$ [S1].

4.  **Construct 3D Vectorial Light Field:**
    The complete generated 3D vectorial light field $\tilde{E}(\rho, z)$ is the coherent summation of all spin-dependent Bessel functions across the full beam array [S1]:
    $$
    \tilde{E}(\rho, z) = \sum_p \sum_q \sum_{m=-N}^{N} \left[ A_{Lp;q;m} |L\rangle + A_{Rp;q;m} |R\rangle \right] J_0(k_{\rho}^{m} |\rho - \rho_{p;q}|) e^{i k_z^m z}
    $$
    where $J_0$ is the zeroth-order Bessel function of the first kind, $\rho_{p;q}$ is the lateral displacement, and the transverse and longitudinal wavevectors $k_{\rho}^m$ and $k_z^m$ satisfy the dispersion relation $(k_{\rho}^m)^2 + (k_z^m)^2 = (k_0)^2$ [S1].

5.  **Determine Metasurface Jones Matrix:**
    Substitute $z=0$ into the field expression to obtain the initial complex field distributions at the metasurface plane for both orthogonal polarization components. From these, determine the unitary Jones matrix profile $\tilde{U}_{meta}(\rho, \phi)$ required to realize the target longitudinal response function [S1]:
    $$
    \tilde{U}_{meta}(\rho, \phi) = R(-\theta(\rho, \phi)) \begin{bmatrix} t_x e^{i\phi_x(\rho,\phi)} & 0 \\ 0 & t_y e^{i\phi_y(\rho,\phi)} \end{bmatrix} R(\theta(\rho, \phi))
    $$
    where $R(\theta)$ is the rotation matrix for local nanopillar orientation $\theta(\rho, \phi)$, and $t_x, t_y, \phi_x, \phi_y$ define the amplitude and phase transmission coefficients for linear polarization along x and y directions [S1].

6.  **Implement via Dual-Matrix Holography:**
    Tailor the geometry ($D_x, D_y$) and in-plane orientation $\theta$ of each rectangular-shaped nanopillar to implement the designed matrix profile $\tilde{U}_{meta}(\rho, \phi)$ using a dual-matrix holography technique, enabling subwavelength control over both phase and polarization at each metasurface pixel [S1].

**Parameters**
*   $Z_0$: Axial extent of the modulated beam array. Determines the discrete spacing of Bessel components $\Delta k_z = 2\pi/Z_0$ [S1].
*   $N$: Half the number of Bessel beam components in the superposition. Total components = $2N + 1$ [S1].
*   $k_0$: Free-space wavevector [S1].

**Edge Cases and Pitfalls**
*   **Normalization:** The weighting coefficients $a_{Lp;q}(z)$ and $a_{Rp;q}(z)$ must satisfy $|a_{Lp;q}(z)|^2 + |a_{Rp;q}(z)|^2 = 1$ to ensure physical realizability [S1].
*   **Orthogonality:** The discrete spacing $\Delta k_z = 2\pi/Z_0$ ensures mutual orthogonality of the Bessel basis components over the defined axial range $Z_0$ [S1].
*   **Dispersion Relation:** The wavevectors $k_{\rho}^m$ and $k_z^m$ must satisfy $(k_{\rho}^m)^2 + (k_z^m)^2 = (k_0)^2$ [S1].

## Operating characteristics
Not given in the sources.

## Background
This method was introduced by Le Tan, Pengcheng Huo, Peicheng Lin, et al. in the paper "Longitudinally engineered metasurfaces for 3D vectorial holography" published in *Light: Science & Applications* in 2026 [S1]. It addresses the limitations of conventional holographic approaches (SLMs, DMDs) and previous metasurface holograms restricted to 2D vectorial holography by achieving full 3D vectorial holography [S1].

## Sources
[S1] Longitudinally engineered metasurfaces for 3D vectorial holography, https://doi.org/10.1038/s41377-025-02158-5, fetched 2026-10-30
