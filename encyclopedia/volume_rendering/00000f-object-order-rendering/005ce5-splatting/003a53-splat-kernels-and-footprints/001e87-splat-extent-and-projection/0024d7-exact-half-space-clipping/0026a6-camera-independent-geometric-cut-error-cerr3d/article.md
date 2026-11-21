# Camera-independent geometric cut error (CErr3D)
Also called: Geometric Cut Error

## Overview
CErr3D is a closed-form diagnostic metric that quantifies the error introduced by approximate half-space clipping operators in Gaussian Splatting by computing the misplaced opacity mass directly from primitive geometry and plane parameters, independent of camera projection or rendering pipeline [S1, unofficial]. It sets itself apart from image-space metrics by evaluating the operator's geometric fidelity against the exact analytic solution, allowing for hardware-agnostic comparison of clipping strategies without the noise of reconstruction artifacts or renderer mismatches [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussians (primitives) and a clipping plane defined by normal $\mathbf{n}$ and offset $\tau$ [S1, unofficial]. Each primitive $i$ is defined by mean $\boldsymbol{\mu}_i$, full covariance $\boldsymbol{\Sigma}_i$, and opacity $\alpha_i$ [S1, unofficial].
*   **Outputs:** A scalar value $\text{CErr}_{3\text{D}} \in [0, 1]$ representing the ratio of total misplaced mass to total kept mass [S1, unofficial].

**Data Structures:**
*   Primitive $i$:
    *   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Mean position [S1, unofficial].
    *   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Full covariance matrix (rotated, including Mip-Splatting filter effects) [S1, unofficial].
    *   $\alpha_i \in [0, 1]$: Opacity [S1, unofficial].
*   Plane:
    *   $\mathbf{n} \in \mathbb{R}^3$: Unit normal vector [S1, unofficial].
    *   $\tau \in \mathbb{R}$: Offset [S1, unofficial].

**Steps:**
1.  For each primitive $i$, compute the normalized distance $t_i$ to the plane:
    $$ t_i = \frac{\tau - \mathbf{n}^\top \boldsymbol{\mu}_i}{\sqrt{\mathbf{n}^\top \boldsymbol{\Sigma}_i \mathbf{n}}} $$
2.  Compute the exact kept opacity mass $e_i$ for primitive $i$ using the standard normal cumulative distribution function (CDF) $\Phi$:
    $$ e_i = \alpha_i \Phi(t_i) $$
3.  Compute the misplaced mass $\epsilon_i$ for primitive $i$ based on the specific clipping operator being evaluated:
    *   **XClipGS (Exact):** $\epsilon_i^{\text{XClip}} = 0$ (by analytic definition) [S1, unofficial].
    *   **Hard Cut (HC):** $\epsilon_i^{\text{HC}} = |\alpha_i \mathbf{1}[t_i \geq 0] - e_i|$ [S1, unofficial].
    *   **Mixture Model (MM):** $\epsilon_i^{\text{MM}} = e_i \Phi \left( -\frac{t_i + \lambda(t_i)}{\sqrt{v(t_i)}} \right)$ [S1, unofficial].
4.  Sum the misplaced masses and kept masses over all primitives:
    $$ \text{CErr}_{3\text{D}} = \frac{\sum_{i=1}^{N} \epsilon_i}{\sum_{i=1}^{N} e_i} $$

**Parameters:**
*   $\Phi(\cdot)$: Standard normal CDF [S1, unofficial].
*   $\mathbf{1}[\cdot]$: Indicator function [S1, unofficial].
*   $\boldsymbol{\Sigma}_i$: Must be the full rotated covariance, not a diagonal approximation, to account for anisotropy and filtering [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The metric is computed in closed form, but finite-precision arithmetic in the evaluation path (e.g., CUDA) can introduce minor deviations from the analytic ideal [S1, unofficial].
*   **Degenerate Input:** If $\mathbf{n}^\top \boldsymbol{\Sigma}_i \mathbf{n} \leq 0$, the denominator in $t_i$ is zero or imaginary [G]. This implies a degenerate covariance (zero variance in the plane normal direction), which should be handled by clamping or skipping the primitive [G].
*   **Boundary:** If $\sum e_i = 0$ (all primitives are fully culled), the metric is undefined [G]. In practice, this case should return NaN or a specific error code [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def standard_normal_cdf(x):
    """Approximation of the standard normal CDF."""
    return 0.5 * (1.0 + math.erf(x / math.sqrt(2.0)))

def compute_cerr3d(primitives, plane_normal, plane_offset, operator_type):
    """
    Compute Camera-independent geometric cut error.
    
    Args:
        primitives: List of dicts with keys 'mu' (3,), 'Sigma' (3,3), 'alpha' (float).
        plane_normal: Array-like (3,) unit normal vector n.
        plane_offset: Float tau.
        operator_type: String, one of 'XClip', 'HC', 'MM'.
        
    Returns:
        float: CErr3D value.
    """
    total_kept_mass = 0.0
    total_misplaced_mass = 0.0
    
    for prim in primitives:
        mu = prim['mu']
        Sigma = prim['Sigma']
        alpha = prim['alpha']
        
        # Compute t_i
        n_Sigma_n = plane_normal @ (Sigma @ plane_normal)
        
        # Handle degenerate covariance
        if n_Sigma_n <= 0:
            # Skip primitives with zero variance along the plane normal [G]
            continue 
            
        t = (plane_offset - plane_normal @ mu) / math.sqrt(n_Sigma_n)
        
        # Exact kept mass
        e_i = alpha * standard_normal_cdf(t)
        total_kept_mass += e_i
        
        # Misplaced mass based on operator
        if operator_type == 'XClip':
            epsilon_i = 0.0
        elif operator_type == 'HC':
            # Hard Cut: 1 if t >= 0, else 0
            indicator = 1.0 if t >= 0 else 0.0
            epsilon_i = abs(alpha * indicator - e_i)
        elif operator_type == 'MM':
            # Mixture Model: Formula requires lambda(t) and v(t)
            # Note: lambda(t) and v(t) are not defined in the provided sources [S1, unofficial].
            # Standard MM formulations typically use truncated normal parameters,
            # but exact definitions are omitted here.
            lambda_t = 0.0  # Placeholder per source omission rule
            v_t = 1.0       # Placeholder per source omission rule
            
            if v_t <= 0:
                epsilon_i = 0.0
            else:
                epsilon_i = e_i * standard_normal_cdf(-(t + lambda_t) / math.sqrt(v_t))
        else:
            raise ValueError(f"Unknown operator type: {operator_type}")
            
        total_misplaced_mass += epsilon_i
        
    if total_kept_mass == 0:
        return float('nan')
        
    return total_misplaced_mass / total_kept_mass
```

## Subtopics

### Difference-referenced cut error (CDE)
A region symmetric difference metric that measures under-removal and over-removal errors, normalized by the reference affected region [S1, unofficial]. It differs from CErr3D by operating in image space and using differencing to cancel reconstruction floor bias [S1, unofficial]. To compute CDE: 1. Render full and clipped versions for both the method ($R$) and reference ($G$) [S1, unofficial]. 2. Compute luminance difference maps $D_R = Y(R_{\text{full}}) - Y(R_{\text{clip}})$ and $D_G = Y(G_{\text{full}}) - Y(G_{\text{clip}})$ [S1, unofficial]. 3. Threshold at $\theta = 0.04$ to obtain binary masks $A_R = \{|D_R| > \theta\}$ and $A_G = \{|D_G| > \theta\}$ [S1, unofficial]. 4. Intersect with a band $\mathcal{B}$ (whole frame for CDEp, $\pm 12$-pixel strip for CDEg) and compute the symmetric difference: $\text{CDE} = \frac{\sum_v |(A_G \setminus A_R) \cap \mathcal{B}| + \sum_v |(A_R \setminus A_G) \cap \mathcal{B}|}{\sum_v |A_G \cap \mathcal{B}|}$ [S1, unofficial]. Pitfall: Direct clipped-vs-reference comparison carries reconstruction floor bias; differencing against unclipped companions cancels this [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** The metric is camera-independent, allowing for offline analysis of clipping operators without rendering [S1, unofficial]. It provides a ground-truth geometric error metric that isolates clipping logic from renderer artifacts [S1, unofficial].
*   **Limitations:** It does not account for image-space perceptual quality or reconstruction errors from the splatting process itself [S1, unofficial]. It requires access to the full primitive geometry and covariance [S1, unofficial].
*   **When to use:** It is used when comparing the geometric fidelity of different half-space clipping algorithms (e.g., XClipGS vs. HC vs. MM) in a controlled, renderer-agnostic manner [S1, unofficial].
*   **When not to use:** It should not be used when image-space perceptual quality is the primary concern; use CDE or Sb in that case [S1, unofficial].

## Background
CErr3D was introduced in the context of "XClipGS: Exact Half-Space Clipping for Medical Volume Gaussian Splatting" [S1, unofficial]. The paper proposes an exact half-space clipping operator that avoids the approximation errors of binary keep/drop projections by using a conditional Gaussian CDF [S1, unofficial]. CErr3D serves as a diagnostic tool to validate the geometric correctness of this and other approximate operators (HC, MM) by quantifying the mass misplaced across the clipping plane [S1, unofficial].

## Sources
[S1] XClipGS: Exact Half-Space Clipping for Medical Volume Gaussian Splatting, https://doi.org/10.48550/arxiv.2608.07760, fetched 2026-10-26 (unofficial)
