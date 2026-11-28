# Pre-computation
## Overview
Pre-computation is a method in volume rendering that calculates intermediate radiance variables and their density derivatives, storing them as voxels to avoid redundant calculations during the ray marching phase [S1]. It solves the problem of expensive volumetric illumination by pre-calculating terms such as incident radiance, absorption coefficients, and scattering sources, allowing the renderer to skip evaluating these complex integrals at each raymarching step [S1]. It is distinguished from full path-tracing methods by its reliance on precomputed segment data and from simple direct illumination by its inclusion of first-order inscattering approximations via a diffusion solver [S1].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Diffusion radiance fields $L_{0d}$, $\vec{L}_{1d}$ and their derivatives; scattering source $S$; extinction coefficient $\kappa$; absorption coefficient $\kappa_a$; total extinction coefficient $\kappa_t$ [S1].
*   **Outputs:** Precomputed radiance field $L_{ri}$ and its derivative $\partial_\pi L_{ri}$; diffusion radiance fields $L_{0d}$, $\vec{L}_{1d}$ and their derivatives $\partial_\pi L_{0d}$, $\partial_\pi \vec{L}_{1d}$ [S1].

**Data Structures:**
The method stores fields as voxels, where each voxel at position $v_{xu}$ contains scalar or vector values for the computed variables [S1]. The specific bit-packing or memory layout is not given in the sources [S1].

**Steps:**
1.  Iterate over each voxel position $v_{xu}$ in the volume in parallel [S1].
2.  Compute the incident radiance $L_{ri}(v_{xu}, \boldsymbol{\omega}')$ and its density derivative $\partial_\pi L_{ri}(v_{xu}, \boldsymbol{\omega}')$ using Equations 31 and 41 [S1].
3.  Compute the intermediate variable $Q_{0ri}(v_{xu})$ and its derivative $\partial_\pi Q_{0ri}(v_{xu})$ using Equations 37 and 15 [S1].
4.  Compute the intermediate variable $\vec{Q}_{1ri}(v_{xu})$ and its derivative $\partial_\pi \vec{Q}_{1ri}(v_{xu})$ using Equations 38 and 16 [S1].
5.  Compute the extinction coefficient $\kappa(v_{xu})$ and its derivative $\partial_\pi \kappa(v_{xu})$ using Equations 35 and 12 [S1].
6.  Compute the scattering source $S(v_{xu})$ and its derivative $\partial_\pi S(v_{xu})$ using Equations 36 and 14 [S1].
7.  Call `DiffusionSolver` to compute the diffusion radiance $L_{0d}$, $\vec{L}_{1d}$ and their derivatives $\partial_\pi L_{0d}$, $\partial_\pi \vec{L}_{1d}$ using the pre-computed $S$, $\kappa$, and $\kappa_a$ [S1].
8.  Return $L_{ri}$, $L_{0d}$, $\vec{L}_{1d}$ and their derivatives [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def Precomputation(L0d, L1d, S, kappa, kappa_a, kappa_t):
    num_voxels = len(S)
    Lri = [None] * num_voxels
    dLri = [None] * num_voxels
    Q0ri = [None] * num_voxels
    dQ0ri = [None] * num_voxels
    Q1ri = [None] * num_voxels
    dQ1ri = [None] * num_voxels
    dkappa = [None] * num_voxels
    dS = [None] * num_voxels

    for vxu in range(num_voxels):
        # Compute incident radiance and derivative (Eq 31, 41) [S1]
        Lri[vxu], dLri[vxu] = compute_Lri_and_derivative(vxu)
        # Compute Q0ri and derivative (Eq 37, 15) [S1]
        Q0ri[vxu], dQ0ri[vxu] = compute_Q0ri_and_derivative(vxu)
        # Compute Q1ri and derivative (Eq 38, 16) [S1]
        Q1ri[vxu], dQ1ri[vxu] = compute_Q1ri_and_derivative(vxu)
        # Compute extinction and derivative (Eq 35, 12) [S1]
        kappa[vxu], dkappa[vxu] = compute_kappa_and_derivative(vxu)
        # Compute scattering source and derivative (Eq 36, 14) [S1]
        S[vxu], dS[vxu] = compute_S_and_derivative(vxu)

    # Solve diffusion radiance and derivatives using Jacobi iteration [S1]
    # [G] Standard Jacobi iteration for linear system is not detailed in sources.
    L0d_out, dL0d_out, L1d_out, dL1d_out = DiffusionSolver(L0d, L1d, S, kappa, kappa_a)
    return Lri, L0d_out, L1d_out, dLri, dL0d_out, dL1d_out

def compute_Lri_and_derivative(vxu):
    # Eq 31, 41 [S1]
    return 0.0, 0.0

def compute_Q0ri_and_derivative(vxu):
    # Eq 37, 15 [S1]
    return 0.0, 0.0

def compute_Q1ri_and_derivative(vxu):
    # Eq 38, 16 [S1]
    return 0.0, 0.0

def compute_kappa_and_derivative(vxu):
    # Eq 35, 12 [S1]
    return 0.0, 0.0

def compute_S_and_derivative(vxu):
    # Eq 36, 14 [S1]
    return 0.0, 0.0

def DiffusionSolver(L0d, L1d, S, kappa, kappa_a):
    # [G] Jacobi iterative method details not provided in sources.
    return L0d, [0.0]*len(L0d), L1d, [0.0]*len(L1d)
```

**Parameters and Edge Cases:**
*   The Monte Carlo method estimates the solid angle integral over the unit sphere $S^2$ for $L_{ri}$ [S1]. The sample count is not given [S1].
*   The `DiffusionSolver` uses a Jacobi iterative method [S1]. The iteration count and convergence criteria are not given [S1].
*   Precision handling for density derivatives $\partial_\pi$ is not detailed [S1].

## Subtopics

### [Local Piecewise Integration for Light Propagation](2365-local-piecewise-integration-for-light-propagation.md)
Approximates global light transport in participating media by decomposing the radiative transfer equation into convection (absorption) and diffusion (scattering) steps on a regular grid. It computes high-resolution local shadows via piecewise opacity segments and propagates light energy using a convection-diffusion solver, enabling real-time rendering via single texture lookups. Choose it when real-time volumetric rendering with accurate local shadows is required.

## Operating characteristics
**Time and Memory Complexity:**
The pre-computation step requires iterating over all voxels in parallel, making it $O(N)$ with respect to the number of voxels $N$ [S1]. The `DiffusionSolver` involves iterative solving, the complexity of which depends on the convergence rate of the Jacobi method [S1]. Memory usage is proportional to the number of voxels, storing multiple fields ($L_{ri}$, $L_{0d}$, $\vec{L}_{1d}$, and their derivatives) [S1].

**Measured Performance:**
Not given in the sources [S1].

**Strengths and Limitations:**
*   **Strengths:** Eliminates redundant calculations during ray marching by storing intermediate variables as voxels [S1]. Allows for efficient reuse of precomputed values [S1].
*   **Limitations:** Requires significant memory to store multiple radiance fields and their derivatives [S1]. The accuracy depends on the pre-computation of incident radiance and the diffusion solver's convergence [S1].

**When to Use:**
Use when rendering participating media where the lighting term is expensive to compute per-voxel during ray marching, and sufficient memory is available to store the precomputed fields [S1].

## Background
**Origin:**
The method is described in "A Differential Diffusion Theory for Participating Media" [S1].

**History and Development:**
The method incorporates a pre-computation step to address challenges in rendering participating media and computing radiance derivatives [S1]. It builds on differential diffusion theory [S1].

## Sources
[S1] A Differential Diffusion Theory for Participating Media, https://doi.org/10.1111/cgf.14956, fetched 2026-10-30
