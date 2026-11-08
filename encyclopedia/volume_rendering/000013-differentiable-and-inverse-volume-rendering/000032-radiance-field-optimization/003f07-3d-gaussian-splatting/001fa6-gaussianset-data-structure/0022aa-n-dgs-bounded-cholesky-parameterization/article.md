# N-DGS Bounded Cholesky Parameterization

Also called: N-DGS

## Overview
N-DGS is a parameterization for the joint covariance of 3D Gaussians that uses bounded Cholesky coefficients multiplied by query diagonal scales to control spatial extent and cross-covariance [S1, unofficial]. It sets itself apart from sibling methods like [dGS Direct Regression Parameterization](2821-dgs-direct-regression-parameterization.md) by implicitly deriving the regression operator from the ambient joint Cholesky factor, rather than optimizing operational quantities directly [S1, unofficial]. This approach avoids the computational cost and instability of inverting the query covariance block during rendering, which is required by methods that explicitly parameterize the regression operator [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** The diagonal scales of the query covariance, $\mathbf{s}_q \in \mathbb{R}^C$; the Cholesky factor of the spatial covariance, $\mathbf{A}$ (lower triangular); and the Cholesky factor of the cross-covariance block, $\mathbf{B}$ [S1, unofficial].
*   **Output:** The joint covariance matrix $\boldsymbol{\Sigma}$, decomposed into blocks $\boldsymbol{\Sigma}_{pp}$ (spatial), $\boldsymbol{\Sigma}_{qq}$ (query), and $\boldsymbol{\Sigma}_{pq}$ (cross) [S1, unofficial].

**Data Structures**
The parameterization relies on the Cholesky decomposition of the joint covariance $\boldsymbol{\Sigma} \in \mathbb{R}^{(p+C) \times (p+C)}$.
Let $\mathbf{L}$ be the lower Cholesky factor such that $\boldsymbol{\Sigma} = \mathbf{L}\mathbf{L}^\top$.
$\mathbf{L}$ is partitioned into blocks corresponding to spatial ($p$) and query ($C$) dimensions:
$$
\mathbf{L} = \begin{pmatrix} \mathbf{A} & \mathbf{0} \\ \mathbf{B} & \mathbf{D} \end{pmatrix}
$$
where:
*   $\mathbf{A} \in \mathbb{R}^{p \times p}$ is lower triangular.
*   $\mathbf{B} \in \mathbb{R}^{C \times p}$ is the cross-covariance Cholesky component.
*   $\mathbf{D} \in \mathbb{R}^{C \times C}$ is lower triangular.

The key innovation is the parameterization of $\mathbf{B}$. Instead of storing $\mathbf{B}$ directly, it is stored as a bounded matrix $\mathbf{B}_e$ scaled by $\mathbf{s}_q$:
$$
\mathbf{B} = \text{diag}(\mathbf{s}_q) \mathbf{B}_e
$$
where $|\mathbf{B}_{e,jk}| \leq 1$ for all entries [S1, unofficial].

**Numbered Steps**
1.  **Retrieve Scales:** Obtain the query diagonal scales $\mathbf{s}_q$ from the Gaussian set. These represent the standard deviations of the query covariance diagonal.
2.  **Reconstruct Cross-Covariance Factor:** Compute $\mathbf{B}$ using the bounded coefficients $\mathbf{B}_e$ and scales $\mathbf{s}_q$:
    $$ \mathbf{B} = \text{diag}(\mathbf{s}_q) \mathbf{B}_e $$
3.  **Compute Spatial Covariance:** Compute $\boldsymbol{\Sigma}_{pp}$ from $\mathbf{A}$:
    $$ \boldsymbol{\Sigma}_{pp} = \mathbf{A}\mathbf{A}^\top $$
4.  **Compute Cross-Covariance:** Compute $\boldsymbol{\Sigma}_{pq}$ using $\mathbf{A}$ and $\mathbf{B}$:
    $$ \boldsymbol{\Sigma}_{pq} = \mathbf{A}\mathbf{B}^\top $$
    *(Note: Since $\boldsymbol{\Sigma}$ is symmetric, $\boldsymbol{\Sigma}_{qp} = \boldsymbol{\Sigma}_{pq}^\top$)*
5.  **Compute Query Covariance:** Compute $\boldsymbol{\Sigma}_{qq}$ from $\mathbf{B}$ and $\mathbf{D}$:
    $$ \boldsymbol{\Sigma}_{qq} = \mathbf{B}\mathbf{B}^\top + \mathbf{D}\mathbf{D}^\top $$
6.  **Assemble Joint Covariance:** Construct the full joint covariance matrix $\boldsymbol{\Sigma}$:
    $$ \boldsymbol{\Sigma} = \begin{pmatrix} \boldsymbol{\Sigma}_{pp} & \boldsymbol{\Sigma}_{pq} \\ \boldsymbol{\Sigma}_{qp} & \boldsymbol{\Sigma}_{qq} \end{pmatrix} $$

**Formulas and Definitions**
*   $\mathbf{s}_q$: Diagonal of the query covariance $\boldsymbol{\Sigma}_{qq}$.
*   $\mathbf{B}_e$: Bounded Cholesky coefficient matrix with entries in $[-1, 1]$.
*   $\boldsymbol{\Sigma}_{pp}$: Spatial covariance block.
*   $\boldsymbol{\Sigma}_{pq}$: Cross-covariance block between spatial and query dimensions.
*   $\boldsymbol{\Sigma}_{qq}$: Query covariance block.

**Bounds and Properties**
The bounded nature of $\mathbf{B}_e$ imposes constraints on the cross-covariance. Specifically:
$$
\|\mathbf{B}\|_2 \leq \|\mathbf{B}\|_F \leq \sqrt{p} \|\mathbf{s}_q\|_2
$$
Consequently, the row-wise norm of the cross-covariance is bounded by:
$$
\|(\boldsymbol{\Sigma}_{pq})_{i,:}\|_2 \leq \sqrt{p} \|\mathbf{s}_q\|_2 \sqrt{(\boldsymbol{\Sigma}_{pp})_{ii}}
$$
[S1, unofficial]

The rendered spatial shift (conditional mean update) is given by:
$$
\Delta \mu_{\text{N-DGS}} = \boldsymbol{\Sigma}_{pq} \boldsymbol{\Sigma}_{qq}^{-1} \delta = \mathbf{M}_{\text{N-DGS}} \delta
$$
where $\delta$ is the query deviation from the mean [S1, unofficial].

A basis-independent bound exists using the normalized cross-covariance $\mathbf{K}$:
$$
\mathbf{K} = \boldsymbol{\Sigma}_{pp}^{-1/2} \boldsymbol{\Sigma}_{pq} \boldsymbol{\Sigma}_{qq}^{-1/2}
$$
With whitened query deviation $\mathbf{h} = \boldsymbol{\Sigma}_{qq}^{-1/2} \delta$, the shift satisfies:
$$
\Delta \mu_{\text{N-DGS}} = \boldsymbol{\Sigma}_{pp}^{1/2} \mathbf{K} \mathbf{h}
$$
$$
\left\| \boldsymbol{\Sigma}_{pp}^{-1/2} \Delta \mu_{\text{N-DGS}} \right\|_2 \leq \left\| \boldsymbol{\Sigma}_{qq}^{-1/2} \delta \right\|_2
$$
This indicates that the shift, measured in marginal spatial standard-deviation units, is bounded by the query deviation in query standard-deviation units [S1, unofficial].

**Parameters**
*   $\mathbf{s}_q$: Query diagonal scales. Typical values depend on the scene scale and query resolution.
*   $\mathbf{B}_e$: Bounded coefficients. Values are constrained to $[-1, 1]$ during optimization or clamping.

**Edge Cases and Pitfalls**
1.  **Interpretation of Off-Diagonals:** Off-diagonal entries of $\mathbf{A}$ contribute to both $\boldsymbol{\Sigma}_{pq}$ and the spatial extent $(\boldsymbol{\Sigma}_{pp})_{ii}$. They should not be interpreted as producing cross-covariance independently of spatial scale [S1, unofficial].
2.  **Ill-Conditioned Query Covariance:** A bound on $\boldsymbol{\Sigma}_{pq}$ alone does not bound the shift $\Delta \mu$. The inverse query covariance $\boldsymbol{\Sigma}_{qq}^{-1}$ and the query deviation $\delta$ also enter the calculation. A narrow or ill-conditioned query covariance can amplify the regression operator, leading to large shifts [S1, unofficial].
3.  **Precision:** The inversion of $\boldsymbol{\Sigma}_{qq}$ during the computation of the shift $\Delta \mu$ can be numerically unstable if $\boldsymbol{\Sigma}_{qq}$ is close to singular. However, N-DGS avoids this inversion during the *parameterization* and *slicing* steps, only requiring it for the final shift calculation if conditional mean is needed [S1, unofficial].

## Subtopics

### N-DGS Covariance-Derived Regression
N-DGS derives the regression operator implicitly from covariance blocks, requiring inversion of the query block to compute the conditional mean [S1, unofficial]. This contrasts with dGS, which directly parameterizes the regression operator $\mathbf{M}$, avoiding this inversion [S1, unofficial]. The off-diagonal entries of the Cholesky factor $\mathbf{A}$ contribute to both cross-covariance and spatial extent, meaning they cannot be interpreted as producing cross-covariance independently of spatial scale [S1, unofficial].

### dGS Exact Implicit Joint Gaussian
This variant refers to the dGS formulation where the joint covariance $\boldsymbol{\Sigma}^*$ is defined by conditional covariance $\boldsymbol{\Sigma}_{\text{cond}}$, regression operator $\mathbf{M}$, and query precision $\boldsymbol{\Pi}$ [S1, unofficial]. The blocks are:
$$
\Sigma_{qq}^* = \frac{1}{2\lambda_o} \mathbf{V}_{qq}^{-1}, \quad \Sigma_{pq}^* = \frac{1}{2\lambda_o} \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}), \quad \Sigma_{pp}^* = \boldsymbol{\Sigma}_{\text{cond}} + \frac{1}{2\lambda_0} \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}) \mathbf{V}_{qq} \text{diag}(\boldsymbol{\Lambda}) \mathbf{V}_{pq}^\top
$$
[S1, unofficial]. The opacity exponent corresponds to the query-marginal quadratic form $-\lambda_o \delta^\top \mathbf{V}_{qq} \delta$ [S1, unofficial]. The coupling strength is controlled by $\boldsymbol{\Lambda} \in [0, 1]^C$, where $\boldsymbol{\Lambda}=0$ decouples position from query and $\boldsymbol{\Lambda}=1$ enables full coupling [S1, unofficial]. The internal factorization of $\mathbf{M}$ into $\mathbf{V}_{pq}$ and $\boldsymbol{\Lambda}$ is not claimed to be bijective [S1, unofficial]. Standard Gaussian splatting uses an unnormalized Gaussian kernel, so the normalization factor is omitted from the renderer [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity**
*   **Memory:** N-DGS stores the Cholesky factors $\mathbf{A}$, $\mathbf{B}_e$, and $\mathbf{D}$. The dimensionality matches the joint covariance: $\frac{p(p+1)}{2} + pC + \frac{C(C+1)}{2}$ parameters [S1, unofficial]. This is equivalent to storing the full joint covariance but with bounded coefficients for $\mathbf{B}$.
*   **Time:** Rendering requires computing the conditional mean shift $\Delta \mu = \boldsymbol{\Sigma}_{pq} \boldsymbol{\Sigma}_{qq}^{-1} \delta$. This involves inverting $\boldsymbol{\Sigma}_{qq}$ ($O(C^3)$) and matrix-vector multiplication. N-DGS avoids inverting the query block during the slicing and parameterization steps, which is a computational advantage over methods that explicitly invert it during optimization [S1, unofficial].

**Measured Performance**
On two trained MCMC scenes with six views each, the opacity-weighted $\|\Delta \mu\|/\bar{s}$ has a median of 0.29–0.30 and a 99th percentile of 4.8–7.0 for N-DGS [S1, unofficial].

**Strengths and Limitations**
*   **Strengths:** Avoids query-block inversion during slicing and parameterization. Provides a bounded parameterization for cross-covariance.
*   **Limitations:** The regression operator is implicit, making it harder to control directly compared to dGS. The shift can be amplified by ill-conditioned query covariance. The bounded Cholesky coefficients do not directly bound the shift, only the cross-covariance norm [S1, unofficial].

**When to Use**
Use N-DGS when computational efficiency in slicing is prioritized and the implicit parameterization is acceptable. It is suitable for scenes where the query covariance is well-conditioned.

**When Not to Use**
Avoid N-DGS if direct control over the regression operator is required, or if the query covariance is likely to be ill-conditioned. In such cases, dGS may be more robust [S1, unofficial].

## Background
N-DGS was introduced in the context of differentiable Gaussian splatting to provide a stable parameterization for joint covariances. It builds on the Cholesky decomposition of the joint covariance matrix. The method was proposed to address the instability of inverting the query covariance block during rendering and optimization [S1, unofficial]. It is part of the family of N-Dimensional Gaussian Splatting methods, which extend 3DGS to higher dimensions (e.g., including time or other attributes) [S1, unofficial].

## Sources
[S1] Direct Conditional Parameterization for N-Dimensional Splatting, https://arxiv.org/abs/2610.22778, fetched 2026-10-30 (unofficial)
