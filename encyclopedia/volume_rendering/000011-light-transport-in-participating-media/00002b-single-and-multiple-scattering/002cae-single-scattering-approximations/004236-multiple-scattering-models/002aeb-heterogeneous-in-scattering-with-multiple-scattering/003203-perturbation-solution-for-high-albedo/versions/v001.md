# Perturbation Solution for High Albedo

Also called: Perturbation expansion, Series of forced conservative scattering equations

## Overview
This method computes the radiance in participating media with high single-scattering albedo ($\beta \approx 1$) by expanding the solution into a series of forced conservative scattering equations. It addresses the failure of standard approximations to model deep shadowing and indirect illumination in highly scattering media like clouds [S1, S2, unofficial]. It differs from [Correction Texture for Surface Albedo](9421-correction-texture-for-surface-albedo.md) by handling volumetric multiple scattering rather than surface reflections, and from [Multiple Scattering via Volume Trace Depth](10698-multiple-scattering-via-volume-trace-depth.md) by using an analytical series expansion rather than stochastic depth tracing.

## How it works
**Inputs:**
- Single-scattering albedo $\beta$ (where $\beta = 1$ is perfect/conservative scattering).
- Source term $q(\mathbf{x})$.
- Phase function $p(\mathbf{x}, \mathbf{n}', \mathbf{n})$.
- Extinction coefficient $\chi(\mathbf{x})$.

**Outputs:**
- Specific intensity $I(\mathbf{x}, \mathbf{n})$.

**Data Structures:**
- Coefficients $Z_k$ (or $I_k$) representing the $k$-th order scattering contribution.
- Spherical harmonic coefficients $I^{lm}(\mathbf{x})$ for the angular expansion.

**Steps:**
1. **Define the Transport Operator:**
   The radiative transfer equation is written in operator notation [S2, unofficial]:
   $$ \hat{L}I + (1 - \beta) \hat{M}I = 0 $$
   where $\hat{L}I = \chi^{-1} \mathbf{n} \cdot \nabla I + I - q/\chi$ and $\hat{M}I = \frac{1}{4\pi} \int p(\mathbf{x}, \mathbf{n}', \mathbf{n}) I(\mathbf{x}, \mathbf{n}') d\Omega'$ [S2, unofficial].

2. **Series Expansion:**
   Expand the intensity $I$ as a power series in $\beta$ (or related parameter $\beta$):
   $$ I = \sum_{k=0}^{\infty} \beta^k I_k $$
   Substituting this into the transport equation and equating powers of $\beta$ yields a sequence of coupled equations [S1, S2, unofficial]:
   $$ \hat{L}I_0 + \hat{M}I_0 = 0 $$
   $$ \hat{L}I_k + \hat{M}I_k = -\hat{M}I_{k-1} \quad \text{for } k \ge 1 $$
   This creates a "series of forced conservative scattering equations" [S1, unofficial].

3. **Solve for the Conservative Case ($I_0$):**
   For perfect scattering ($\beta=1$), the radiation field is diffuse. Expand $I_0$ in spherical harmonics [S2, unofficial]:
   $$ I_0(\mathbf{x}, \mathbf{n}) = \sum_{l=0}^{\infty} \sum_{m=-l}^{l} I^{lm}(\mathbf{x}) Y_{lm}(\theta, \phi) $$
   Truncate the series at $l=1$ (p-wave) for computer graphics applications [S2, unofficial]. Substitute this expansion into the $k=0$ equation to obtain a system of coupled first-order differential equations for the coefficients $I^{lm}(\mathbf{x})$ [S2, unofficial]:
   $$ \sum_{l,m} \chi^{-1} \nabla I^{lm}(\mathbf{x}) \cdot \langle Y_{l'm'} | \mathbf{n} | Y_{lm} \rangle - I^{lm}(\mathbf{x}) \delta_{ll'} \delta_{mm'} + \chi^{-1} q + I^{lm}(\mathbf{x}) \langle Y_{l'm'} | p | Y_{lm} \rangle = 0 $$
   Here $\langle X | O | Y \rangle = \int X^* O Y \, d\Omega$ [S2, unofficial]. Solve this system analytically or numerically to find $I_0$ [S2, unofficial].

4. **Compute Corrections ($I_k$):**
   Use the previously computed $I_{k-1}$ as the source term for the equation for $I_k$. Repeat until the desired order is reached [S1, S2, unofficial].

5. **Reconstruct Intensity:**
   Sum the contributions:
   $$ I = \sum_{k=0}^{\infty} \beta^k I_k $$
   (Note: [S1, unofficial] uses notation $Z = \sum \beta^k Z_k$) [S1, unofficial].

**Parameters:**
- Truncation order $k$: Determined by required accuracy and $\beta$.
- Spherical harmonic truncation $l$: Typically $l=1$ (p-wave) [S2, unofficial].

**Edge Cases:**
- If $\beta$ is not close to 1, the series may converge slowly or diverge. The formal criterion for the Neumann series expansion is that the spectral radius of $\hat{M}$ must be less than 1 [S2, unofficial].

## Subtopics

### High-Albedo Approximation
This variant specifically targets the "abnormally dark" shadowing artifacts seen in standard ray tracing for clouds with high albedo [S1, unofficial]. It treats the medium as nearly conservative, solving the $I_0$ case exactly (via spherical harmonics) and perturbing from there [S1, S2, unofficial]. Choose this when modeling thick, highly scattering volumes like clouds where single-scattering approximations fail to show subsurface illumination [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of scattering orders computed and the cost of solving the coupled differential equations for $I^{lm}$. Solving the conservative case analytically is efficient, but higher orders require iterative source updates [S1, S2, unofficial].
- **Memory Complexity:** Requires storage for spherical harmonic coefficients $I^{lm}$ and intermediate scattering terms.
- **Strengths:** Accurately models multiple scattering and shadowing in high-albedo media where other methods fail [S1, unofficial].
- **Limitations:** Complex to implement due to spherical harmonic expansions and coupled differential equations. Convergence is not guaranteed if $\beta$ is too low [S2, unofficial].

## Background
Originated from the need to accurately render clouds, which are scattering media with very high albedo [S1, unofficial]. Blinn (1982) suggested treating multiple scattering in this context [S1, unofficial]. The method is detailed in Shirley and Tuchman's work on volume rendering algorithms and further analyzed in mathematical models of volume rendering [S2, S1, unofficial].

## Sources
[S1] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
[S2] Volume Rendering Mathematical Models and Algorithmic Aspects, https://core.ac.uk/works/16970577, fetched 2026-10-29 (unofficial)
