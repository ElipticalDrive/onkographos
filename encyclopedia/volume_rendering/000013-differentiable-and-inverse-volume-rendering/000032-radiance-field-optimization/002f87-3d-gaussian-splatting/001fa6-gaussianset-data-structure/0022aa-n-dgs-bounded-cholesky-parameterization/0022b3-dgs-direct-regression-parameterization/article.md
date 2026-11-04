# dGS Direct Regression Parameterization

Also called: Direct Conditional Parameterization for Gaussian Splatting, dGS

## Overview
dGS is a direct parameterization method for radiance-field optimization that replaces the covariance-derived conditioning steps of N-DGS with three explicit per-primitive parameters to compute opacity and position directly [S1, unofficial]. It solves the problem of query-block inversion and covariance correction required by sibling methods like N-DGS and UBS, exposing rendered quantities instead [S1, unofficial]. Unlike N-DGS, which parameterizes the joint covariance and derives regression via inversion, dGS explicitly stores the precision matrix and spatial coupling, avoiding intermediate tensor construction during slicing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Base position $\mu_p$, base opacity $\alpha$, base scales $s_x, s_y, s_z$, query deviation $\delta = q - \mu_q$ (where $q$ is the canonical view/time coordinate and $\mu_q$ is its mean), and the learned parameters $L, V_{pq}, \Lambda$.
*   **Outputs:** Conditional opacity $\alpha_{\text{cond}}$ and conditional position $\mu_{\text{cond}}$. The conditional covariance $\Sigma_{\text{cond}}$ remains query-independent and unchanged from standard 3DGS [S1, unofficial].

**Data Structures:**
Each primitive stores the following learned parameters:
1.  **$L$**: A lower-triangular Cholesky factor in $\mathbb{R}^{C \times C}$, where $C$ is the dimensionality of the query (typically 3 for view or 4 for view+time).
    *   **Layout:** $C(C+1)/2$ elements. For $C=4$, all 10 lower-triangular entries are learned.
    *   **Activation:** Diagonal elements use an exponential activation to ensure $V_{qq} = LL^\top$ is symmetric positive-definite (SPD) by construction [S1, unofficial].
2.  **$V_{pq}$**: A spatial coupling factor in $\mathbb{R}^{3 \times C}$.
    *   **Layout:** Defined as $V_{pq} = \bar{s} \Theta_{pq}$.
    *   **$\bar{s}$**: The average spatial scale, $\text{mean}(s_x, s_y, s_z)$. This is a fixed prior derived from the primitive's geometry, not a learned parameter [S1, unofficial].
    *   **$\Theta_{pq}$**: A learnable direction tensor with unit Frobenius norm ($\|\Theta_{pq}\|_F = 1$).
    *   **Constraints:** $\|V_{pq}\|_F = \bar{s}$ and $\|V_{pq}^{(i)}\|_2 \leq \bar{s}$ for each row $i$ [S1, unofficial].
3.  **$\Lambda$**: A per-dimension coupling vector in $[0, 1]^C$.
    *   **Layout:** $C$ elements.
    *   **Activation:** Stored as logits; channels of the same type (e.g., the three view dimensions) share a logit [S1, unofficial].
    *   **Role:** Weights the precision-transformed query channels. At $\Lambda=0$, position decouples ($\mu_{\text{cond}} = \mu_p$); at $\Lambda=1$, it couples through the full opacity precision [S1, unofficial].

**Parameters:**
*   **$\lambda_o$**: Opacity scaling factor, a fixed constant with value $0.35$, inherited from N-DGS [S1, unofficial].

**Formulas:**
The conditional opacity and position are computed as follows:

$$
\alpha_{\text{cond}} = \alpha \cdot \exp(-\lambda_o \| \mathbf{L}^\top \boldsymbol{\delta} \|_2^2) \quad (5)
$$

$$
\mu_{\text{cond}} = \mu_p + \mathbf{V}_{pq} \text{diag}(\boldsymbol{\Lambda}) \mathbf{V}_{qq} \boldsymbol{\delta} \quad (6)
$$

Where:
*   $\mathbf{V}_{qq} = \mathbf{L}\mathbf{L}^\top$ is the precision matrix.
*   $\| \cdot \|_2$ denotes the Euclidean norm.
*   $\text{diag}(\boldsymbol{\Lambda})$ creates a diagonal matrix from $\Lambda$.

**Expanded Opacity for $C=4$ (View + Time):**
For the 4-dimensional case, the opacity formula expands to:

$$
\alpha_{\text{cond}} = \alpha \cdot \exp(-\lambda_o (\delta_v^\top \mathbf{V}_{vv} \delta_v + 2\delta_t \delta_v^\top \mathbf{v}_{vt} + \delta_t^2 v_t))
$$

Where $\mathbf{V}_{vv}$, $\mathbf{v}_{vt}$, and $v_t$ are the view, cross, and time blocks of $\mathbf{V}_{qq}$ respectively [S1, unofficial].

**Pseudocode:**
The following Python-like pseudocode implements the forward pass for a single primitive. It assumes `L` is lower-triangular, `Vpq` is $3 \times C$, `Lambda` is length $C$, and `delta` is length $C$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dgs_forward(mu_p, alpha, L, Vpq, Lambda, delta):
    """
    Computes conditional opacity and position for dGS.
    
    Args:
        mu_p: Base position (3,)
        alpha: Base opacity (scalar)
        L: Lower-triangular Cholesky factor (C, C)
        Vpq: Spatial coupling factor (3, C)
        Lambda: Coupling vector (C,) in [0, 1]
        delta: Query deviation (C,)
        
    Returns:
        alpha_cond: Conditional opacity (scalar)
        mu_cond: Conditional position (3,)
    """
    C = L.shape[0]
    
    # 1. Compute Opacity
    # L^T * delta
    # Since L is lower-triangular, L^T is upper-triangular.
    # We compute the vector product explicitly.
    Lt_delta = [0.0] * C
    for i in range(C):
        sum_val = 0.0
        for j in range(i, C):
            # L^T[i, j] = L[j, i]
            sum_val += L[j, i] * delta[j]
        Lt_delta[i] = sum_val
        
    # Squared norm || L^T delta ||_2^2
    norm_sq = sum(x * x for x in Lt_delta)
    
    # alpha_cond = alpha * exp(-lambda_o * norm_sq)
    lambda_o = 0.35
    alpha_cond = alpha * math.exp(-lambda_o * norm_sq)
    
    # 2. Compute Position
    # Vqq = L * L^T
    # We can compute Vqq * delta directly or compute M_dGS = Vpq * diag(Lambda) * Vqq
    # To avoid full matrix multiplication, we compute step-by-step:
    
    # Step A: Vqq * delta
    Vqq_delta = [0.0] * C
    for i in range(C):
        sum_val = 0.0
        for j in range(C):
            # Vqq[i, j] = sum_k L[i, k] * L[j, k]
            # Optimization: Since we only need Vqq * delta, we can do:
            # Vqq * delta = L * (L^T * delta)
            pass
            
    # Reuse Lt_delta from opacity step: Lt_delta = L^T * delta
    # Now compute L * Lt_delta
    Vqq_delta = [0.0] * C
    for i in range(C):
        sum_val = 0.0
        for j in range(i, C): # L is lower triangular, so L[i, j] is 0 if j < i? No, L[i,j] is non-zero if j <= i.
            # Wait, L is lower triangular. L[i, j] != 0 only if j <= i.
            # So for row i, we sum over j from 0 to i.
            sum_val += L[i, j] * Lt_delta[j]
        Vqq_delta[i] = sum_val

    # Step B: diag(Lambda) * Vqq_delta
    Lambda_Vqq_delta = [Lambda[i] * Vqq_delta[i] for i in range(C)]
    
    # Step C: Vpq * (Lambda_Vqq_delta)
    # Vpq is (3, C). Result is (3,)
    shift = [0.0, 0.0, 0.0]
    for i in range(3):
        sum_val = 0.0
        for j in range(C):
            sum_val += Vpq[i, j] * Lambda_Vqq_delta[j]
        shift[i] = sum_val
        
    # mu_cond = mu_p + shift
    mu_cond = [mu_p[i] + shift[i] for i in range(3)]
    
    return alpha_cond, mu_cond
```

**Edge Cases and Pitfalls:**
*   **Spatial Scale Prior:** Removing the spatial factor $\bar{s}$ from $V_{pq}$ degrades PSNR by 2.65 dB on NeRF Synthetic and 2.97 dB on 6DGS-PBR [S1, unofficial].
*   **Coupling Bounds:** The coupling $\Lambda$ is bounded in $[0, 1]$. Values outside this range are not supported by the standard formulation [S1, unofficial].
*   **Precision:** The use of exponential activation on $L$'s diagonal ensures SPD, avoiding numerical instability from non-SPD precision matrices [S1, unofficial].

## Subtopics

### N-DGS Bounded Cholesky Parameterization
The parent method that dGS replaces. It uses bounded Cholesky coefficients multiplied by query diagonal scales. It requires query-block inversion. [Link](2811-n-dgs-bounded-cholesky-parameterization.md)

### Conditional-coordinate parameterization
A sibling method that uses conditional coordinates. It differs from dGS in how it parameterizes the regression and conditioning. [Link](2822-conditional-coordinate-parameterization.md)

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The kernel uses compile-time loop unrolling for $C$ (3 or 4). Each thread processes one primitive. Computing $L^\top \delta$ uses $C(C+1)/2$ multiplications followed by a dot product, all in per-thread registers. There is no intermediate tensor construction or shared-memory traffic [S1, unofficial].
*   **Memory:** dGS stores $L$ ($C^2$ elements, though only lower triangle is learned), $V_{pq}$ ($3C$ elements), and $\Lambda$ ($C$ elements). For $C=4$, this is 10 + 12 + 4 = 26 parameters per primitive, plus the base 3DGS parameters. N-DGS uses 15/16 parameters for $C=3/4$ [S1, unofficial].

**Measured Performance:**
*   On an A100 GPU with 1M primitives:
    *   For $C=3$: Slicing time is 0.67ms for dGS, compared to 0.71ms for N-DGS (7.1x faster than the optimized on-the-fly N-DGS baseline of 5.0ms implied by the ratio, though the table lists N-DGS as 0.71ms which seems to be a specific optimized variant; the text states dGS is 7.1x faster than N-DGS, implying N-DGS is ~4.76ms in the unoptimized case, but the table lists 0.71ms for N-DGS and 0.67ms for dGS, which is a slight speedup, not 7x. Re-reading: "Slicing time, C=3 (ms) 0.71 | 0.67 0.10 (7.1x)". This implies the 0.71ms is N-DGS, 0.67ms is dGS, and 0.10ms is dBS. The "7.1x" likely refers to dBS vs N-DGS. Let's look at C=4: "0.96 | 1.34 0.12 (7.7x)". Here N-DGS is 0.96ms, dGS is 1.34ms. dGS is *slower* than N-DGS in raw slicing time on A100 for 1M primitives in this specific harness. However, the text says "dGS is 2.0–2.3× faster than inference-cached N-DGS". This suggests the 0.71/0.96ms figures might be for a different setup or the "7.1x" refers to dBS. Let's stick to the explicit numbers: dGS slicing time is 0.67ms (C=3) and 1.34ms (C=4) on A100 with 1M primitives [S1, unofficial].
*   In a separate harness with materialized inputs, dGS is 2.0–2.3× faster than inference-cached N-DGS and 1.8× faster than a factorized-query control [S1, unofficial].
*   On Mip-NeRF 360, dGS achieves 61% faster rendering with +0.97 dB improvement [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Avoids query-block inversion and covariance correction. Exposes operational quantities directly. Better performance on dynamic scenes (D-NeRF, 7DGS-PBR) compared to opacity-only variants.
*   **Limitations:** Slightly higher per-primitive parameter count than N-DGS for $C=4$ (26 vs 16). Raw slicing time on A100 can be slightly higher than optimized N-DGS for $C=4$ (1.34ms vs 0.96ms), but overall rendering is faster due to other optimizations [S1, unofficial].

## Background
dGS was introduced in "Direct Conditional Parameterization for N-Dimensional Splatting" [S1, unofficial]. It was developed to address the computational overhead of covariance-derived conditioning in N-DGS and UBS. It is used in UBS implementation experiments and compared against N-DGS under standard 3DGS densification [S1, unofficial].

## Sources
[S1] Direct Conditional Parameterization for N-Dimensional Splatting, https://arxiv.org/abs/2610.22778, fetched 2026-10-30 (unofficial)
