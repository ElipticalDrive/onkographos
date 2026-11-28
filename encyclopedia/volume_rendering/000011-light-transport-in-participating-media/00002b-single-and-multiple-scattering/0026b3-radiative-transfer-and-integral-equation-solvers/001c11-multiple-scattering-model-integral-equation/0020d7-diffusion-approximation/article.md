# Diffusion Approximation

Also called: Classical Diffusion Approximation, Diffusion Approximation for Multiple Scattering, Diffusion Approximation of RTE, Diffusion Method (Stam), Diffusion Theory, Radiative Diffusion Equation, Stam [93, 94], Stam diffusion approximation

## Overview
The Diffusion Approximation is a method for computing multiple scattering in participating media by replacing the computationally expensive integral form of the Radiative Transfer Equation (RTE) with a partial differential equation (PDE) [S1, S3]. It models the medium radiance as a function of the first two angular moments of the radiation field: the scalar irradiance (fluence) and the vector irradiance (flux) [S1, S4]. Unlike [Full Propagator Approximation for Multiply-Scattered Radiance](1298-full-propagator-approximation-for-multiply-scattered-radianc.md) or [Monte Carlo Method for Multiple Scattering](1940-monte-carlo-method-for-multiple-scattering.md), which track angular distribution or stochastic paths, the diffusion approximation assumes that in highly scattering media, light distribution becomes nearly isotropic, allowing angular dependence to be smoothed out [S3, S4]. It is particularly efficient for dense, high-opacity media where photons undergo long random walks [S1].

## How it works
**Inputs:**
*   Medium properties: Absorption coefficient $\sigma_a(\mathbf{x})$, scattering coefficient $\sigma_s(\mathbf{x})$, extinction coefficient $\sigma_t(\mathbf{x}) = \sigma_a(\mathbf{x}) + \sigma_s(\mathbf{x})$ [S1].
*   Source terms: Incident unscattered light field $q_{ri}(\mathbf{x})$ and self-emission field $j(\mathbf{x})$ [S1].
*   Domain: Volume $\Omega$ with boundary $\partial \Omega$ [S1].

**Outputs:**
*   Fluence field $\phi(\mathbf{x})$ [S1].
*   Flux field $\mathbf{E}(\mathbf{x})$ [S1].
*   Approximated medium radiance $L_m(\mathbf{x}, \boldsymbol{\omega})$ [S1].

**Data Structures:**
*   The method typically operates on a grid or a set of blobs representing the volume [S2, S3].
*   For grid-based implementations, $\phi$ and $\mathbf{E}$ are stored as scalar and vector fields at grid nodes or cell centers [G].
*   For blob-based implementations (e.g., Stam's method), a system of linear equations is formed for each blob, solving for coefficients $J_0$ and $J_1$ [S2].

**Formulas:**
1.  **Radiance Approximation:**
    The medium radiance is approximated by the first two terms of the angular expansion [S1, S4]:
    $$ L_m(\mathbf{x}, \boldsymbol{\omega}) \approx \frac{1}{4\pi} \phi(\mathbf{x}) + \frac{3}{4\pi} \boldsymbol{\omega} \cdot \mathbf{E}(\mathbf{x}) $$
    where $\phi(\mathbf{x})$ is the fluence (scalar irradiance) and $\mathbf{E}(\mathbf{x})$ is the flux (vector irradiance) [S1, S4].

2.  **Fick's First Law:**
    The flux is related to the gradient of the fluence via the diffusion coefficient $D(\mathbf{x})$ [S1]:
    $$ \mathbf{E}(\mathbf{x}) = -D(\mathbf{x}) \nabla \phi(\mathbf{x}) $$
    where the diffusion coefficient is defined as:
    $$ D(\mathbf{x}) = \frac{1}{3\sigma_t(\mathbf{x})} $$

3.  **Radiative Diffusion Equation (Fick's Second Law):**
    Substituting Fick's law into the energy conservation equation yields the governing PDE for fluence [S1]:
    $$ \nabla \cdot (D(\mathbf{x}) \nabla \phi(\mathbf{x})) = \sigma_a(\mathbf{x}) \phi(\mathbf{x}) - q_{ri}(\mathbf{x}) - j(\mathbf{x}) $$
    Here, $q_{ri}(\mathbf{x})$ is the source field from reduced incident light, and $j(\mathbf{x})$ is the emission source [S1].

4.  **Total Radiance at Ray Endpoint:**
    The final radiance at a point $\mathbf{x}_e$ along a ray is computed by combining the attenuated incident light and the in-scattered light from the diffusion solution [S1]:
    $$ L(\mathbf{x}_e, \boldsymbol{\omega}) = L(\mathbf{x}_s, \boldsymbol{\omega}) T(\mathbf{x}_s, \mathbf{x}_e) + \Delta x \sum_n T(\mathbf{x}_n, \mathbf{x}_e) Q(\mathbf{x}_n) $$
    where $T$ is transmittance, $\mathbf{x}_s$ is the start point, and $Q(\mathbf{x}_n)$ is the source term derived from the diffusion approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_diffusion_approximation(grid, cell_size, sigma_a, sigma_s, q_ri, j):
    sigma_t = sigma_a + sigma_s
    D = 1.0 / (3.0 * sigma_t)
    N = grid.num_cells()
    
    # 1. Discretize PDE: -div(D * grad(phi)) + sigma_a * phi = q_ri + j [G]
    A = build_sparse_matrix(D, sigma_a, grid, cell_size)
    b = q_ri + j
    apply_dirichlet_boundary_conditions(A, b, grid, phi=0.0)
    
    # 2. Solve linear system for fluence [S1, S2]
    if N < 1000:
        phi = solve_lu_decomposition(A, b)
    else:
        phi = solve_relaxation_scheme(A, b, max_iterations=1000)
        
    # 3. Compute flux via Fick's law [S1]
    E = -D * compute_gradient(phi, grid, cell_size)
    
    # 4. Ray march to compute final radiance [S1]
    radiance = 0.0
    transmittance = 1.0
    for sample_point in ray_samples:
        radiance += transmittance * (sigma_a[sample_point] * phi[sample_point] - q_ri[sample_point] - j[sample_point]) * cell_size
        transmittance *= exp(-sigma_t[sample_point] * cell_size)
        
    return phi, E, radiance
```

**Algorithm Steps:**
1.  **Precompute Source Terms:** Calculate $q_{ri}(\mathbf{x})$ from the incident unscattered light and $j(\mathbf{x})$ from self-emission [S1].
2.  **Discretize the Diffusion Equation:**
    *   *Grid-based:* Discretize the PDE $\nabla \cdot (D \nabla \phi) = \sigma_a \phi - q_{ri} - j$ using finite differences or finite elements to form a linear system $A\phi = b$ [G].
    *   *Blob-based:* Form a system of linear equations for each blob, restricting the source radiance to $J_m(\boldsymbol{\omega}) = J_0 + J_1 \cdot \boldsymbol{\omega}$ [S2].
3.  **Solve for Fluence:**
    *   Apply boundary conditions (e.g., Dirichlet: $\phi = 0$ on $\partial \Omega$) [S1].
    *   Solve the linear system for $\phi(\mathbf{x})$.
        *   If the number of unknowns $v < 1000$, use direct LU decomposition [S2].
        *   If $v$ is larger, use an iterative relaxation scheme (convergence not guaranteed) [S2].
4.  **Compute Flux:** Calculate $\mathbf{E}(\mathbf{x}) = -D(\mathbf{x}) \nabla \phi(\mathbf{x})$ using the solved fluence field [S1].
5.  **Render:** Compute the final radiance $L(\mathbf{x}, \boldsymbol{\omega})$ using the radiance approximation formula and ray marching [S1].

**Parameters:**
*   The method assumes isotropic scattering or requires an effective scattering coefficient $\sigma_s' = \sigma_s(1-g)$ if anisotropy is present [S4].
*   The method is limited to volume scenes without surface models in some implementations [S3].

**Edge Cases and Pitfalls:**
*   **Vacuum/Low Extinction:** The method breaks down in regions of low extinction ($\sigma_t \to 0$) or vacuum, where $D(\mathbf{x})$ diverges [S1]. This violates the physical constraint $|\mathbf{E}| \leq \phi$ [S1].
*   **Thin Features:** Dipole variants of diffusion approximation tend to overestimate absorption by thin features (e.g., edges of a nose or helmet) [S4].
*   **Complex Scenes:** For scenes with many surfaces, the cost of progressive shooting of energy from surface patches can be expensive, requiring hierarchical approaches [S2].

## Subtopics

### [Flux-Limited Diffusion](3239-flux-limited-diffusion.md)
Modifies the classical diffusion coefficient to be flux-limited and fluence-dependent, handling heterogeneous media and vacuum regions where classical diffusion fails. Choose this when rendering scenes with sharp opacity transitions or low-extinction zones.

### [Diffusion Approximation for Cloud Radiance](9971-diffusion-approximation-for-cloud-radiance.md)
Specifically discretizes the diffusion equation for volumetric cloud rendering, using multigrid relaxation to solve for the isotropic radiance component. Choose this for large-scale atmospheric or cloud volumes where standard grid solvers are too memory-intensive.

## Operating characteristics
*   **Time Complexity:** Solving the linear system is generally faster than path tracing for dense media [S2]. Blob methods scale with the number of blobs $v$, requiring $O(v^3)$ for direct LU decomposition when $v < 1000$, or iterative relaxation for larger $v$ [S2].
*   **Memory Complexity:** Uses far less memory than grid-based full RTE solvers [S2].
*   **Strengths:** Very fast for dense, highly scattering media; includes both double and multiple scattering in a single model [S4].
*   **Limitations:** Inaccurate in low-scattering regions, near boundaries, or for thin features; assumes near-isotropic radiance [S1, S4].
*   **When to Use:** Dense participating media (e.g., fog, skin, milk) where multiple scattering dominates [S1, S4].
*   **When Not to Use:** Sparse media, vacuum regions, or scenes requiring accurate single-scattering angular details [S1, S4].

## Background
The diffusion approximation originates from the observation that in highly scattering media, the angular distribution of light becomes isotropic due to multiple scattering events [S3, S4]. It was applied to volume rendering by Stam [S2] to solve multiple scattering between blobs modeling media, offering a significant speedup over path tracing. It is widely used in computer graphics for rendering subsurface scattering and dense participating media [S1, S4].

## Sources
[S1] Flux-Limited Diffusion for Multiple Scattering in Participating Media, https://archive.org/details/arxiv-1403.8105, fetched 2026-10-26
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S3] A Differential Diffusion Theory for Participating Media, https://doi.org/10.1111/cgf.14956, fetched 2026-10-30
[S4] Precomputed Multiple Scattering for Rapid Light Simulation in Participating Media, https://doi.org/10.1109/TVCG.2018.2890466, fetched 2026-10-26
