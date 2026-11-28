# Anisotropic Radiative Transfer with SGGX Phase Functions

Also called: Jakob et al. anisotropic RTE, Heitz et al. SGGX model

## Overview
This method extends the standard Radiative Transfer Equation (RTE) to model participating media with structured micro-geometries, such as fabrics and fur, by making scattering parameters direction-dependent [S1]. It differs from isotropic models by using a micro-flake normal distribution function $D(m)$ to derive anisotropic extinction and phase functions, and specifically employs the SGGX representation for $D(m)$ to enable fast evaluation and linear interpolation of phase functions [S1].

## How it works
**Inputs:**
*   Micro-flake density $\rho$ [S1].
*   Directionally independent single-scattering albedo $\alpha$ [S1].
*   SGGX structure tensor $S$ (a $3 \times 3$ symmetric positive-definite matrix) defining the normal distribution [S1].
*   Incident radiance field $L(\omega')$ [S1].

**Outputs:**
*   Outgoing radiance $L(\omega)$ [S1].
*   Extinction coefficient $\sigma_t(\omega)$ [S1].
*   Scattering coefficient $\sigma_s(\omega)$ [S1].
*   Phase function $f(\omega' \rightarrow \omega)$ [S1].

**Data Structures:**
*   **Structure Tensor $S$**: A $3 \times 3$ symmetric matrix. In implementation, store as 6 floats (upper triangular) or 9 floats. The determinant $|S|$ and inverse $S^{-1}$ are computed from this.
*   **Radiance $L$**: A function or sampled values over the sphere $S^2$.

**Formulas:**
1.  **Anisotropic RTE**:
    $$ (\omega \cdot \nabla)L(\omega) = -\sigma_t(\omega)L(\omega) + \sigma_s(\omega) \int_{S^2} L(\omega') f(\omega' \rightarrow \omega) d\omega' $$
    where $\omega$ is the outgoing direction, $\nabla$ is the spatial gradient, and the source term is neglected for non-emissive media [S1].

2.  **Scattering Parameters**:
    *   Projected Area:
        $$ \sigma(\omega) = \int_{S^2} \langle \omega, \omega' \rangle D(\omega') d\omega' $$
        [S1]
    *   Extinction Coefficient:
        $$ \sigma_t(\omega) = \rho \sigma(\omega) $$
        [S1]
    *   Scattering Coefficient:
        $$ \sigma_s(\omega) = \alpha \sigma_t(\omega) $$
        [S1]

3.  **Phase Function**:
    $$ f(\omega \rightarrow \omega') = \frac{1}{\sigma(\omega')} \int_{S^2} p(m, \omega' \rightarrow \omega) \langle \omega', m \rangle D(m) dm $$
    where $p(m, \omega' \rightarrow \omega)$ is the scattering profile for a micro-flake with normal $m$ [S1].

4.  **SGGX Normal Distribution**:
    $$ D(m) := \frac{1}{\pi |S| (m^T S^{-1} m)^2} $$
    where $|S|$ is the determinant of $S$ [S1].

**Parameters:**
*   $\alpha$: Single-scattering albedo, ratio of scattering to extinction coefficient ($\alpha = \sigma_s / \sigma_t$) [S1].
*   $\rho$: Micro-flake density [S1].

**Steps:**
1.  Compute the determinant $|S|$ and inverse $S^{-1}$ of the input structure tensor $S$.
2.  For a given direction $\omega$, compute the projected area $\sigma(\omega)$ by integrating $\langle \omega, \omega' \rangle D(\omega')$ over the sphere. Note: The sources define $\sigma(\omega)$ via an integral but do not provide a closed-form analytical solution for the SGGX projected area in the provided text. [G: In practice, $\sigma(\omega)$ for SGGX is often computed via numerical integration or specific approximations derived from the tensor properties, but the exact formula is not given in the sources.]
3.  Compute $\sigma_t(\omega) = \rho \sigma(\omega)$.
4.  Compute $\sigma_s(\omega) = \alpha \sigma_t(\omega)$.
5.  To evaluate the phase function $f(\omega' \rightarrow \omega)$, compute the integral involving $D(m)$ and the scattering profile $p$. The sources define this integral but do not provide a closed-form evaluation for $p$ or the final phase function value. [G: The SGGX phase function is typically evaluated by decomposing it into lobes or using specific sampling strategies, but the exact evaluation algorithm is not detailed in the provided notes.]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sggx_scattering_params(S, rho, alpha, omega):
    """
    Computes scattering parameters for a given direction omega using SGGX.
    
    Args:
        S: 3x3 symmetric positive-definite matrix (Structure Tensor)
        rho: float, micro-flake density
        alpha: float, single-scattering albedo
        omega: 3D vector, outgoing direction
        
    Returns:
        sigma_t: float, extinction coefficient
        sigma_s: float, scattering coefficient
    """
    # 1. Compute determinant and inverse of S
    det_S = determinant(S)
    inv_S = inverse(S)
    
    # 2. Compute projected area sigma(omega)
    # The source defines sigma(omega) as an integral over the sphere.
    # No closed-form is provided in the sources.
    # Implementation requires numerical integration or a specific approximation.
    # sigma_omega = integrate_projected_area(S, omega)
    
    # Placeholder for the missing integral evaluation [G]
    # In a real implementation, this would involve integrating D(m) * <omega, m>
    # over the hemisphere or sphere.
    sigma_omega = evaluate_projected_area_integral(S, omega)
    
    # 3. Compute extinction coefficient
    sigma_t = rho * sigma_omega
    
    # 4. Compute scattering coefficient
    sigma_s = alpha * sigma_t
    
    return sigma_t, sigma_s

def evaluate_projected_area_integral(S, omega):
    """
    Placeholder for the integral: integral_{S^2} <omega, omega'> D(omega') domega'
    where D(m) = 1 / (pi * |S| * (m^T S^-1 m)^2)
    """
    # This function is not defined in the sources.
    # A common approach is to use the properties of the SGGX distribution
    # to derive an analytical or semi-analytical solution, or use Monte Carlo.
    # Since the source does not provide this, we cannot implement it from text alone.
    raise NotImplementedError("Projected area integral for SGGX not provided in sources.")
```

**Edge Cases and Pitfalls:**
*   **Singularity**: If $S$ is singular (determinant is zero), $D(m)$ is undefined. Ensure $S$ is positive-definite [S1].
*   **Isotropic Limit**: When $D$ is uniform, the model collapses to the standard isotropic RTE. This is a check for correctness [S1].
*   **Interpolation**: Independently interpolating density and albedo yields poor results. Interpolation should be handled carefully, specifically for the structure tensor $S$ [S1].

## Subtopics

### SGGX Phase Function Interpolation
To represent the average of $n$ SGGX phase functions determined by matrices $S_1, S_2, \dots, S_n$ with one SGGX function with matrix $\tilde{S}$, use the arithmetic average of those matrices: $\tilde{S} = \frac{1}{n} \sum_{i=1}^n S_i$ [S1]. Previously, independently interpolating density and single-scattering albedo was shown to yield poor results [S1].

## Operating characteristics
*   **Complexity**: The SGGX representation offers "fast evaluation" [S1]. The interpolation of phase functions is linear in the structure tensor $S$ [S1].
*   **Performance**: The method is capable of closely capturing the appearance of complex materials such as fabrics and fur [S1].
*   **Strengths**: Fast evaluation, easy linear interpolation of phase functions, physically-based micro-flake model [S1].
*   **Limitations**: The sources do not provide explicit time/memory complexity bounds or measured performance numbers. The evaluation of the projected area integral $\sigma(\omega)$ and the phase function integral is not explicitly detailed, which may impact performance in practice [S1].

## Background
The Radiative Transfer Equation (RTE) was originally proposed by Chandrasekhar [1960] for random media with disorganized microstructures [S1]. Jakob et al. [2010] generalized the RTE to capture directional dependencies of scattering parameters, modeling media as a collection of two-sided small mirrors (micro-flakes) with a statistical normal distribution $D$ [S1]. This framework has been used to model fabrics and fur [S1]. Heitz et al. [2015] introduced the SGGX-based representation for the micro-flake normal distribution, offering fast evaluation and easy interpolation [S1].

## Sources
[S1] Downsampling scattering parameters for rendering anisotropic media, https://doi.org/10.1145/2980179.2980228, fetched 2026-10-28
