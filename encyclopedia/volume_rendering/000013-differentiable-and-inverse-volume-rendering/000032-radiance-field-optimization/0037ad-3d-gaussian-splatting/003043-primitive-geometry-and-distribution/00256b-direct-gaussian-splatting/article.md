# Direct Gaussian Splatting

Also called: dGS, direct conditional parameterization for Gaussian kernel

## Overview
Direct Gaussian Splatting (dGS) is a method for N-dimensional splatting that replaces the covariance-derived conditioning step of standard N-DGS with explicit per-primitive parameters, removing the need for per-primitive matrix inversions and regression products during rendering [S1, unofficial]. It solves the computational bottleneck of recovering conditional mean, opacity, and covariance from a joint distribution by directly parameterizing the opacity precision and position regression operator [S1, unofficial]. Unlike N-DGS, which couples position and opacity through shared covariance blocks requiring expensive linear algebra, dGS separates these effects using a Cholesky precision factor for opacity and a factorized query-to-position regression operator with learnable per-dimension coupling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ primitives. Each primitive $i$ is defined by the base parameters from [Primitive Geometry and Distribution](6228-primitive-geometry-and-distribution.md) (mean $\boldsymbol{\mu}_p$, base opacity $\alpha$, etc.) plus the conditioning variables:
    *   Canonical query mean $\boldsymbol{\mu}_q$ (e.g., view direction or time).
    *   Cholesky precision factor $\mathbf{L}$ for opacity.
    *   Spatial coupling factor $\mathbf{V}_{pq}$.
    *   Per-dimension coupling vector $\boldsymbol{\Lambda}$.
    *   Precision matrix block $\mathbf{V}_{qq}$ (shared with opacity).
*   **Output:** Conditional mean $\boldsymbol{\mu}_{\text{cond}}$ and conditional opacity $\alpha_{\text{cond}}$ for a given query $\mathbf{q}$ (e.g., camera view direction).

**Data Structures:**
*   $\mathbf{L}$: Lower triangular Cholesky factor of the opacity precision matrix. Size $C \times C$ where $C$ is the dimension of the query space (typically 3 for view direction).
*   $\mathbf{V}_{pq}$: Spatial coupling factor. Size $3 \times C$. Constrained such that $\|\mathbf{V}_{pq}\|_F = \bar{s}$, where $\bar{s}$ is the mean scale of the primitive.
*   $\boldsymbol{\Lambda}$: Per-dimension coupling vector. Size $C$. Controls the interpolation between query-independent position and full covariance coupling.
*   $\mathbf{V}_{qq}$: Precision matrix block for the query space. Size $C \times C$. Symmetric positive-definite.

**Numbered Steps:**
1.  **Compute Query Deviation:** Calculate the deviation of the query from the canonical mean:
    $$ \boldsymbol{\delta} = \mathbf{q} - \boldsymbol{\mu}_q $$
2.  **Compute Opacity:** Calculate the conditional opacity using the Cholesky factor $\mathbf{L}$:
    $$ \alpha_{\text{cond}} = \alpha \cdot \exp(-\lambda_0 \|\mathbf{L}^T \boldsymbol{\delta}\|_2^2) $$
    where $\lambda_0$ is a scaling factor.
3.  **Compute Position Shift:** Calculate the shift to the conditional mean using the factorized regression operator:
    $$ \boldsymbol{\mu}_{\text{cond}} = \boldsymbol{\mu}_p + \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}) \mathbf{V}_{qq} \boldsymbol{\delta} $$
    Note: The term $\mathbf{V}_{qq} \boldsymbol{\delta}$ is computed first, then scaled element-wise by $\boldsymbol{\Lambda}$, then multiplied by $\mathbf{V}_{pq}$.

**Formulas:**
*   **Opacity:**
    $$ \alpha_{\text{cond}} = \alpha \cdot \exp(-\lambda_0 \|\mathbf{L}^T \boldsymbol{\delta}\|_2^2) $$
    [S1, unofficial]
*   **Position:**
    $$ \boldsymbol{\mu}_{\text{cond}} = \boldsymbol{\mu}_p + \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}) \mathbf{V}_{qq} \boldsymbol{\delta} $$
    [S1, unofficial]
*   **Coupling Interpretation:**
    When $\boldsymbol{\Lambda} = \mathbf{0}$, the position is query-independent (recovering dGS-O). When $\boldsymbol{\Lambda} = \mathbf{1}$, position and opacity share the full precision $\mathbf{V}_{qq}$, mirroring N-DGS coupling [S1, unofficial].

**Parameters:**
*   $\lambda_0$: Scaling factor for the opacity kernel. Default value is 0.35 [S1, unofficial].
*   $\mathbf{L}$: Learnable Cholesky precision factor per primitive [S1, unofficial].
*   $\mathbf{V}_{pq}$: Learnable spatial coupling factor per primitive [S1, unofficial].
*   $\boldsymbol{\Lambda}$: Learnable per-dimension coupling vector per primitive [S1, unofficial].
*   $\mathbf{V}_{qq}$: Learnable precision matrix block per primitive [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method avoids matrix inversion, improving numerical stability compared to N-DGS [S1, unofficial].
*   **Boundary:** The coupling vector $\boldsymbol{\Lambda}$ allows smooth interpolation between decoupled and fully coupled states [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_gaussian_splatting_conditioning(
    q: Vector,           # Query vector (e.g., view direction)
    mu_q: Vector,        # Canonical query mean
    mu_p: Vector,        # Base position mean
    alpha: float,        # Base opacity
    L: Matrix,           # Cholesky precision factor (C x C)
    Vpq: Matrix,         # Spatial coupling factor (3 x C)
    Lambda: Vector,      # Per-dimension coupling (C)
    Vqq: Matrix,         # Precision matrix block (C x C)
    lambda_0: float = 0.35 # Scaling factor
) -> Tuple[Vector, float]:
    """
    Computes conditional opacity and mean for a direct Gaussian splatting primitive.
    
    Args:
        q: Query vector (e.g., view direction).
        mu_q: Canonical query mean.
        mu_p: Base position mean.
        alpha: Base opacity.
        L: Cholesky precision factor for opacity.
        Vpq: Spatial coupling factor.
        Lambda: Per-dimension coupling vector.
        Vqq: Precision matrix block.
        lambda_0: Scaling factor for opacity kernel.
        
    Returns:
        mu_cond: Conditional mean position.
        alpha_cond: Conditional opacity.
    """
    # Step 1: Compute query deviation
    delta = q - mu_q
    
    # Step 2: Compute opacity
    # alpha_cond = alpha * exp(-lambda_0 * ||L^T * delta||^2)
    L_T_delta = L.T @ delta
    opacity_term = lambda_0 * dot(L_T_delta, L_T_delta)
    alpha_cond = alpha * exp(-opacity_term)
    
    # Step 3: Compute position shift
    # mu_cond = mu_p + Vpq * diag(Lambda) * Vqq * delta
    
    # Compute Vqq * delta
    Vqq_delta = Vqq @ delta
    
    # Apply per-dimension coupling: diag(Lambda) * (Vqq * delta)
    # This is element-wise multiplication of Lambda and Vqq_delta
    coupled_shift = Lambda * Vqq_delta
    
    # Apply spatial coupling: Vpq * coupled_shift
    position_shift = Vpq @ coupled_shift
    
    # Final conditional mean
    mu_cond = mu_p + position_shift
    
    return mu_cond, alpha_cond
```

## Subtopics

### Direct Beta Splatting
Direct Beta Splatting (dBS) is a variant of dGS that swaps the Gaussian opacity decay for a Beta opacity decay, following the UBS implementation. It reuses the direct conditional structure ($\mathbf{L}$, $\mathbf{V}_{pq}$, $\mathbf{V}_{qq}$) but derives coupling from bandwidth. Unlike dGS, which uses a separate learned coupling $\boldsymbol{\Lambda}$ for position, dBS derives the coupling $\boldsymbol{\Lambda}_\beta$ from the opacity bandwidth $\boldsymbol{\beta}$. It is chosen when a Beta kernel is preferred for its specific decay properties, while retaining the computational benefits of direct conditioning. Link: [Direct Beta Splatting](4088-direct-beta-splatting.md) (Note: No separate article exists, details below).

**Definition:** A variant of dGS that swaps the Gaussian opacity decay for a Beta opacity decay, following the UBS implementation.

**Steps:**
1.  Compute $z = \mathbf{L}^T \boldsymbol{\delta}$.
2.  For each dimension $i$, compute $(1 - \tanh(z_i^2))^{\beta_i}$.
3.  Multiply these terms together and scale by base opacity $\alpha$.
4.  Compute position: $\boldsymbol{\mu}_{\text{cond}} = \boldsymbol{\mu}_p + \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}_\beta) \mathbf{V}_{qq} \boldsymbol{\delta}$.

**Formulas:**
*   **Beta Opacity:**
    $$ \alpha_{\text{cond}} = \alpha \cdot \prod_{i=1}^C (1 - \tanh(z_i^2))^{\beta_i} $$
    where $z = \mathbf{L}^T \boldsymbol{\delta}$ and $\beta_i = 4 \exp(\beta_{\text{raw}_i})$.
*   **Beta Position:**
    $$ \boldsymbol{\mu}_{\text{cond}} = \boldsymbol{\mu}_p + \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}_\beta) \mathbf{V}_{qq} \boldsymbol{\delta} $$
    where $\boldsymbol{\Lambda}_\beta = \min(\boldsymbol{\beta}/4, 1)$.

**Parameters:**
*   $\beta_i$: Bandwidth parameter for Beta kernel, defined as $\beta_i = 4 \exp(\beta_{\text{raw}_i})$.
*   $\boldsymbol{\Lambda}_\beta$: Coupling vector for dBS, derived from bandwidth as $\boldsymbol{\Lambda}_\beta = \min(\boldsymbol{\beta}/4, 1)$.

**Number of Conditioning Parameters:**
*   For $C=3$, UBS and dBS use 19 conditioning parameters; for $C=4$, they use 27.
*   N-DGS uses 15/16 and dGS uses 22/24 for $C=3/4$ respectively.

**Comparison:**
dGS uses a separate learned coupling $\boldsymbol{\Lambda}$ for position, while dBS derives coupling $\boldsymbol{\Lambda}_\beta$ from the opacity bandwidth $\boldsymbol{\beta}$. Both retain the same direct conditional structure.

## Operating characteristics
*   **Time Complexity:** Removes per-primitive matrix inversions and regression products required by N-DGS, reducing computational cost per query [S1, unofficial].
*   **Memory Complexity:** Stores explicit parameters $\mathbf{L}$, $\mathbf{V}_{pq}$, $\boldsymbol{\Lambda}$, and $\mathbf{V}_{qq}$ per primitive. For $C=3$, dGS uses 22 parameters per primitive, while N-DGS uses 15 [S1, unofficial].
*   **Measured Performance:** Ablation studies show that removing the spatial scale prior $\bar{s}$ degrades PSNR by 2.65 dB on NeRF Synthetic [S1, unofficial].
*   **Strengths:** Removes expensive operations, separates position from opacity coupling, and improves numerical stability by avoiding matrix inversion [S1, unofficial].
*   **Limitations:** Uses more parameters than N-DGS for $C=3$ (22 vs 15) [S1, unofficial].
*   **When to Use:** When computational efficiency during rendering is critical and view-dependent effects are needed [S1, unofficial].
*   **When Not to Use:** When memory is extremely constrained and the parameter count of dGS is prohibitive compared to N-DGS [S1, unofficial].

## Background
Direct Gaussian Splatting was introduced in the paper "Direct Conditional Parameterization for N-Dimensional Splatting" [S1, unofficial]. It builds on the N-DGS framework [S1, unofficial] and the UBS method [S1, unofficial]. The method was developed to address the computational bottleneck of covariance-derived conditioning in N-dimensional splatting [S1, unofficial].

## Sources
[S1] Direct Conditional Parameterization for N-Dimensional Splatting, https://arxiv.org/abs/2610.22778, fetched 2026-10-30 (unofficial)
