# BRDF Smoothness Loss

Also called: BRDF Smoothness Regularization, Smoothness Loss (BRDF), Smoothness Priors for BRDF

## Overview
BRDF Smoothness Loss is a regularization term applied during material field optimization to enforce spatial coherence in BRDF parameters such as roughness, albedo, and normals [S2]. It penalizes drastic changes between neighboring surface points based on the assumption that BRDF properties do not change drastically in homogeneous areas [S1]. The method differs from standard smoothness priors by incorporating edge-aware weighting based on pseudo-albedo differences, which suppresses the loss across material boundaries [S2].

## How it works
The method computes a loss value based on the relative difference of BRDF parameters between a reference point and a perturbed neighbor, weighted by the difference in pseudo-albedo.

**Inputs:**
*   $\mathbf{x}_{\text{pos}}_k$: Position of the $k$-th sampled point on the surface [S2].
*   $\boldsymbol{\beta}(\mathbf{x})$: Vector of BRDF parameters at position $\mathbf{x}$ [S2].
*   $\mathbf{c}(\mathbf{x}, \mathbf{d}_{\text{out}})$: Rendered color at position $\mathbf{x}$ for outgoing direction $\mathbf{d}_{\text{out}}$ [S2].
*   $\mathbf{c}_{\text{irradiance}}(\mathbf{x}, \mathbf{d}_{\text{out}})$: Irradiance color at position $\mathbf{x}$, computed by setting the BRDF to perfectly Lambertian with albedo 1 [S2].
*   $\boldsymbol{\xi}$: Perturbation vector [S2].
*   $C_{\text{BRDF}}$: Loss weight coefficient [S2].
*   $\epsilon$: Perturbation scale [S2].
*   $w_k$: Sampling weight for point $k$ [S2].

**Outputs:**
*   $\mathcal{L}_{\text{BRDF}}$: Scalar loss value [S2].

**Data Structures:**
*   $\boldsymbol{\beta}(\mathbf{x})$: Generic parameter vector containing BRDF properties. [S1] defines separate losses for roughness ($R$), normal ($N$), and albedo ($B$). [S2] treats $\boldsymbol{\beta}$ as a unified vector.

**Steps:**
1.  Sample a perturbation vector $\boldsymbol{\xi}$ from a normal distribution $\mathcal{N}(\mathbf{0}, \epsilon \mathrm{I})$ [S2].
2.  Compute the pseudo-albedo at the reference point $\mathbf{x}_{\text{pos}}_k$ and the perturbed point $\mathbf{x}_{\text{pos}}_k + \boldsymbol{\xi}$:
    $$ \mathbf{a}_{\text{pseudo}}(\mathbf{x}) = \frac{\mathbf{c}(\mathbf{x}, \mathbf{d}_{\text{out}})}{\mathbf{c}_{\text{irradiance}}(\mathbf{x}, \mathbf{d}_{\text{out}})} $$
    [S2]
3.  Compute the edge-aware weight $\lambda$ based on the absolute difference of pseudo-albedos:
    $$ \lambda(\mathbf{x}_{\text{pos}}_k, \mathbf{x}_{\text{pos}}_k + \boldsymbol{\xi}) = | \mathbf{a}_{\text{pseudo}}(\mathbf{x}_{\text{pos}}_k) - \mathbf{a}_{\text{pseudo}}(\mathbf{x}_{\text{pos}}_k + \boldsymbol{\xi}) | $$
    [S2]
4.  Compute the relative difference of the BRDF parameters $\boldsymbol{\beta}$:
    $$ \Delta \boldsymbol{\beta} = \left| \frac{\boldsymbol{\beta}(\mathbf{x}_{\text{pos}}_k) - \boldsymbol{\beta}(\mathbf{x}_{\text{pos}}_k + \boldsymbol{\xi})}{\operatorname{max}(\boldsymbol{\beta}(\mathbf{x}_{\text{pos}}_k), \boldsymbol{\beta}(\mathbf{x}_{\text{pos}}_k + \boldsymbol{\xi}))} \right| $$
    [S2]
    *Note: Division and max operations are element-wise. A small constant is added to the denominator to prevent division by zero if parameters are uninitialized or clamped to zero [G].*
5.  Accumulate the loss over all sampled points $k$:
    $$ \mathcal{L}_{\text{BRDF}} = C_{\text{BRDF}} \sum_{k} w_k \cdot \Delta \boldsymbol{\beta} \cdot \lambda(\mathbf{x}_{\text{pos}}_k, \mathbf{x}_{\text{pos}}_k + \boldsymbol{\xi}) $$
    [S2]

**Alternative Formulation (S1):**
[S1] proposes a gradient-based edge-aware smoothness prior. For roughness $R$:
$$ \mathcal{L}_{s, r} = \Vert \nabla R \Vert \exp (-\Vert \nabla C_{gt} \Vert ) $$
[S1]
where $R = \sum_{i \in N} w_i r_i$ is the rendered roughness map [S1]. Similar constraints $\mathcal{L}_{s,n}$ and $\mathcal{L}_{s,b}$ are defined for normal and albedo [S1].

**Parameters:**
*   $\epsilon$: Standard deviation of the perturbation noise. Set to 0.01 for the TensoIR dataset, 0.005 for other datasets [S2].
*   $C_{\text{BRDF}}$: Loss coefficient. Set to 0.05 for the TensoIR dataset, 0.001 for other datasets [S2].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** The denominator $\operatorname{max}(\boldsymbol{\beta}(\mathbf{x}_{\text{pos}}_k), \boldsymbol{\beta}(\mathbf{x}_{\text{pos}}_k + \boldsymbol{\xi}))$ can be zero if BRDF parameters are uninitialized or clamped to zero. A small constant is added to the denominator [G].
*   **Boundary Conditions:** The perturbed point $\mathbf{x}_{\text{pos}}_k + \boldsymbol{\xi}$ may fall outside the valid surface or volume. Out-of-bounds queries must be handled by returning zero or clamping coordinates [G].

## Operating characteristics
Time and memory complexity, measured performance, scaling behavior, strengths, limitations, and usage conditions are not given in the sources.

## Background
The concept of BRDF smoothness priors originates from the observation that BRDF properties do not change drastically in homogeneous areas [S1]. The specific formulation using pseudo-albedo weighting and relative differences is a variant of TensoIR’s smoothness loss [S2]. The gradient-based exponential weighting variant is used in Relightable 3D Gaussians [S1].

## Sources
[S1] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
[S2] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
