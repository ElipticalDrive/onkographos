# dGS Direct Regression Parameterization

Also called: Direct Conditional Parameterization for Gaussian Splatting, dGS

## Overview
dGS is a parameterization for 3D Gaussian Splatting that computes conditional opacity and position directly from explicit per-primitive parameters, avoiding the query-block inversion and covariance correction required by N-DGS [S1, unofficial]. It replaces the joint covariance-derived conditioning with a Cholesky factor for opacity precision, a spatial coupling factor for position, and a coupling vector to tie them, enabling faster slicing and differentiable optimization of Gaussian positions without local minima [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Base Gaussian parameters (mean $\boldsymbol{\mu}_p$, base opacity $\alpha$, spatial scales $s_x, s_y, s_z$), query deviation $\boldsymbol{\delta} = \boldsymbol{q} - \boldsymbol{\mu}_q$ (where $\boldsymbol{q}$ is the query coordinate in view/time space), and learned conditioning parameters ($\mathbf{L}, \mathbf{V}_{pq}, \boldsymbol{\Lambda}$).
*   **Outputs:** Conditional opacity $\alpha_{\text{cond}}$, conditional mean $\boldsymbol{\mu}_{\text{cond}}$. The conditional covariance $\boldsymbol{\Sigma}_{\text{cond}}$ remains query-independent and is computed as in standard 3DGS.

**Data Structures:**
1.  **$\mathbf{L}$ (Cholesky Factor):** Lower-triangular matrix in $\mathbb{R}^{C \times C}$, where $C$ is the dimensionality of the query space (e.g., $C=3$ for view, $C=4$ for view+time). Diagonal elements use exponential activation to ensure $\mathbf{V}_{qq} = \mathbf{L}\mathbf{L}^\top$ is symmetric positive-definite. For $C=4$, all 10 lower-triangular entries are learned.
2.  **$\mathbf{V}_{pq}$ (Spatial Coupling Factor):** Matrix in $\mathbb{R}^{3 \times C}$. Defined as $\mathbf{V}_{pq} = \bar{s} \boldsymbol{\Theta}_{pq}$, where $\bar{s} = \text{mean}(s_x, s_y, s_z)$ is the average spatial scale, and $\boldsymbol{\Theta}_{pq}$ is a learnable direction with unit Frobenius norm ($\|\boldsymbol{\Theta}_{pq}\|_F = 1$). This enforces $\|\mathbf{V}_{pq}\|_F = \bar{s}$ and $\|\mathbf{V}_{pq}^{(i)}\|_2 \leq \bar{s}$ [S1, unofficial].
3.  **$\boldsymbol{\Lambda}$ (Coupling Vector):** Vector in $[0, 1]^C$. Weights the precision-transformed query channels in the position update. Channels of the same type (e.g., the three view dimensions) share a logit.

**Formulas:**
*   **Opacity Calculation:**
    $$ \alpha_{\text{cond}} = \alpha \cdot \exp(-\lambda_o \|\mathbf{L}^\top \boldsymbol{\delta}\|_2^2) $$
    where $\lambda_o = 0.35$ is a fixed opacity scaling factor inherited from N-DGS [S1, unofficial].

*   **Position Calculation:**
    $$ \boldsymbol{\mu}_{\text{cond}} = \boldsymbol{\mu}_p + \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}) \mathbf{V}_{qq} \boldsymbol{\delta} $$
    where $\mathbf{V}_{qq} = \mathbf{L}\mathbf{L}^\top$ is the precision matrix [S1, unofficial].

*   **Expanded Opacity for $C=4$ (View + Time):**
    $$ \alpha_{\text{cond}} = \alpha \cdot \exp(-\lambda_o (\boldsymbol{\delta}_v^\top \mathbf{V}_{vv} \boldsymbol{\delta}_v + 2\delta_t \boldsymbol{\delta}_v^\top \mathbf{v}_{vt} + \delta_t^2 v_t)) $$
    where $\mathbf{V}_{vv}, \mathbf{v}_{vt}, v_t$ are the view, cross, and time blocks of $\mathbf{V}_{qq}$ [S1, unofficial].

*   **Matrix Definitions:**
    $$ \mathbf{V}_{qq} = \mathbf{L}\mathbf{L}^\top $$
    $$ \mathbf{M}_{\text{dGS}} = \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}) \mathbf{V}_{qq} $$
    $$ \mathbf{F}_{pq} = \bar{s} \boldsymbol{\Theta}_{pq}, \quad \|\boldsymbol{\Theta}_{pq}\|_F = 1 $$

**Parameters:**
*   $\lambda_o$: Opacity scaling factor, fixed at 0.35 [S1, unofficial].
*   $\bar{s}$: Explicit spatial-scale prior on the coupling factor, calculated as the mean of the primitive's spatial scales [S1, unofficial].

**Pitfalls:**
*   Removing the spatial factor $\bar{s}$ degrades PSNR by 2.65 dB on NeRF Synthetic and 2.97 dB on 6DGS-PBR [S1, unofficial].
*   $\bar{s}$ is a prior on the coupling factor, not a bound on the effective regression or rendered shift, which also depends on $\text{diag}(\boldsymbol{\Lambda})\mathbf{V}_{qq}$ and $\boldsymbol{\delta}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dgs_conditioning(mu_p, alpha, s_x, s_y, s_z, L, V_pq, Lambda, delta, lambda_o=0.35):
    """
    Computes conditional opacity and position for a single Gaussian primitive.
    
    Args:
        mu_p: Base position (3,)
        alpha: Base opacity (scalar)
        s_x, s_y, s_z: Spatial scales (scalars)
        L: Lower-triangular Cholesky factor (C, C)
        V_pq: Spatial coupling factor (3, C)
        Lambda: Coupling vector (C,) with values in [0, 1]
        delta: Query deviation (C,)
        lambda_o: Opacity scaling factor (scalar)
        
    Returns:
        alpha_cond: Conditional opacity (scalar)
        mu_cond: Conditional mean (3,)
    """
    C = L.shape[0]
    
    # 1. Compute Precision Matrix V_qq = L * L^T
    # Since L is lower-triangular, we can compute V_qq explicitly or use L^T delta directly.
    # For opacity, we need ||L^T delta||^2.
    
    # Compute L^T * delta
    # L^T is upper triangular.
    L_T_delta = [0.0] * C
    for i in range(C):
        sum_val = 0.0
        for j in range(i, C):
            sum_val += L[j, i] * delta[j]
        L_T_delta[i] = sum_val
        
    # Compute squared norm for opacity
    norm_sq = sum(x*x for x in L_T_delta)
    
    # 2. Compute Conditional Opacity
    alpha_cond = alpha * math.exp(-lambda_o * norm_sq)
    
    # 3. Compute Conditional Position
    # mu_cond = mu_p + V_pq * diag(Lambda) * V_qq * delta
    
    # Step 3a: Compute V_qq * delta
    # V_qq = L * L^T. We can compute V_qq explicitly or compute L * (L^T * delta).
    # Let temp = L^T * delta (already computed as L_T_delta)
    # Then V_qq * delta = L * temp
    
    V_qq_delta = [0.0] * C
    for i in range(C):
        sum_val = 0.0
        for j in range(i, C):
            sum_val += L[i, j] * L_T_delta[j]
        V_qq_delta[i] = sum_val
        
    # Step 3b: Apply diag(Lambda)
    Lambda_V_qq_delta = [Lambda[i] * V_qq_delta[i] for i in range(C)]
    
    # Step 3c: Multiply by V_pq
    # V_pq is (3, C), Lambda_V_qq_delta is (C,)
    # Result is (3,)
    V_pq_term = [0.0] * 3
    for i in range(3):
        sum_val = 0.0
        for j in range(C):
            sum_val += V_pq[i, j] * Lambda_V_qq_delta[j]
        V_pq_term[i] = sum_val
        
    # Step 3d: Add to mu_p
    mu_cond = [mu_p[i] + V_pq_term[i] for i in range(3)]
    
    return alpha_cond, mu_cond
```

## Subtopics

### [Probabilistic Prediction of Pixel-Aligned Gaussians](6424-probabilistic-prediction-of-pixel-aligned-gaussians.md)
A method for predicting Gaussian parameters from per-pixel features using a discrete probability distribution over depth buckets, solving local minima issues in direct regression. It provides a ~1 dB PSNR boost over direct regression baselines by enabling differentiable optimization of Gaussian positions via a reparameterization trick.

## Operating characteristics
*   **Time Complexity:** The kernel uses compile-time loop unrolling for $C$ (3 or 4). Each thread processes one primitive. $\mathbf{L}^\top \boldsymbol{\delta}$ uses $C(C+1)/2$ multiplications followed by a dot product, all in per-thread registers. No intermediate tensors or shared-memory traffic [S1, unofficial].
*   **Memory Complexity:** Stores $\mathbf{L}$ ($C^2$ params), $\mathbf{V}_{pq}$ ($3C$ params), and $\boldsymbol{\Lambda}$ ($C$ params). For $C=3$, 22 parameters; for $C=4$, 24 parameters [S1, unofficial].
*   **Measured Performance:** On an A100 with 1M primitives, slicing time for $C=3$ is 0.67ms for dGS (7.1x faster than N-DGS at 0.71ms) and 1.34ms for $C=4$ (7.7x faster than N-DGS at 0.96ms) [S1, unofficial]. In a separate harness, dGS is 2.0–2.3× faster than inference-cached N-DGS [S1, unofficial].
*   **Strengths:** Avoids query-block inversion, regression construction, and covariance correction. Faster than N-DGS. Preserves compatibility with existing densification strategies as $\boldsymbol{\Sigma}_{\text{cond}}$ is query-independent [S1, unofficial].
*   **Limitations:** Performance gain is most pronounced on Mip-NeRF 360. On 6DGS-PBR, dGS-O (opacity-only) shows modest speedup but quality drop, while dGS recovers quality [S1, unofficial].
*   **When to Use:** When faster slicing is required and differentiable position optimization is needed without the complexity of joint covariance parameterization.
*   **When Not to Use:** When strict geometric bounds on conditional covariance are required (N-DGS provides this via joint Cholesky) or when using standard MCMC densification with fixed coupling (dGS-O may underperform) [S1, unofficial].

## Background
dGS was introduced in "Direct Conditional Parameterization for N-Dimensional Splatting" (2026) as a direct alternative to N-DGS [S1, unofficial]. It was developed to address the computational overhead of query-block inversion in N-DGS and UBS, exposing rendered quantities directly through explicit parameters [S1, unofficial]. It is used in UBS implementation experiments and compared against N-DGS under standard 3DGS densification [S1, unofficial].

## Sources
[S1] Direct Conditional Parameterization for N-Dimensional Splatting, https://arxiv.org/abs/2610.22778, fetched 2026-10-30 (unofficial)
