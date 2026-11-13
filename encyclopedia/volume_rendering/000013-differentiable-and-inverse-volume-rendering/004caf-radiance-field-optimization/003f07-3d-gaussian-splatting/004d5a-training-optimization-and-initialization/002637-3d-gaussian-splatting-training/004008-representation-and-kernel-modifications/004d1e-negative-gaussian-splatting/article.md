# Negative Gaussian Splatting

Also called: Diff-Gaussian

## Overview
Negative Gaussian Splatting (NegGS) extends the standard 3D Gaussian Splatting representation by incorporating "negative Gaussians"—ellipsoids with negative color values—into the optimization process [S1, S2]. This allows the model to represent high-frequency elements, complex nonlinear structures, and rapid color transitions that are difficult to capture with standard positive-only Gaussians, while potentially reducing the total number of primitives required [S1, S2]. Unlike standard Gaussian Splatting, which relies on many components to approximate complex shapes, NegGS uses the cancellation effect between positive and negative Gaussians to achieve greater flexibility in shape and color modulation [S1, S2].

## How it works
NegGS modifies the primitive definition and optimization strategy of 3D Gaussian Splatting. It introduces a joint family of Gaussians that includes both positive and negative color components, effectively implementing a "Diff-Gaussian" distribution where the effective density is modeled by the ratio of two Gaussian probability density functions (PDFs) [S1, S2].

### Inputs and Outputs
*   **Inputs:** A set of camera poses and images; initial 3D point cloud (typically derived from Structure-from-Motion); hyperparameters for initialization (specifically the ratio of negative Gaussians).
*   **Outputs:** A set of Gaussian primitives $G_{NeGS}$, each defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and color $\mathbf{c}$ (where $\mathbf{c}$ can be negative), optimized to minimize rendering loss.

### Data Structures
The method leverages the original Gaussian Splatting framework implementation [S1]. To preserve rendering speed and compatibility with the differentiable rasterization pipeline, negative Gaussians are tracked via a **binary mask** [S1].
*   **Primitive Tuple:** Each Gaussian $i$ retains the standard attributes: $\boldsymbol{\mu}_i \in \mathbb{R}^3$, $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (or its rotation/scale decomposition), $\alpha_i \in [0, 1]$, and $\mathbf{c}_i \in \mathbb{R}^3$.
*   **Color Constraint:** In NegGS, $\mathbf{c}_i$ is allowed to take negative values [S1].
*   **Mask:** A binary flag or mask distinguishes whether a Gaussian belongs to the positive set $G_{GS}$ or the negative set $G_{-GS}$ [S1].

### The Diff-Gaussian Concept
The theoretical basis for NegGS is the "Diff-Gaussian" distribution, defined as the ratio of two Gaussian PDFs [S2]. Let $f_{m_0, \Sigma_0}$ and $f_{m_1, \Sigma_1}$ be two Gaussian densities. The Diff-Gaussian density is proportional to:
$$ f_{Diff}(w) \propto \frac{f_{m_0, \Sigma_0}(w)}{f_{m_1, \Sigma_1}(w)} $$
This ratio allows for the approximation of complex structures (e.g., donut or moon-shaped datasets) that standard Gaussians cannot represent efficiently [S2].

### Optimization Strategy
The optimization operates on a joint family of Gaussians:
$$ G_{NeGS} = G_{GS} \cup G_{-GS} $$
where $G_{GS}$ contains tuples with positive color values $c_i$ and $G_{-GS}$ contains tuples with negative color values $c_i$ [S1].

1.  **Initialization:** The initial number of negative Gaussians is a critical hyperparameter. Experiments indicate that setting the number of negative Gaussians to approximately **20%** of the total Gaussians yields optimal efficacy [S1]. The initial number of negative components is significantly less than that of positive components [S1].
2.  **Rendering:** The rendering pipeline remains identical to standard 3D Gaussian Splatting, using alpha blending. The negative colors allow for destructive interference (cancellation) with positive Gaussians, effectively subtracting color/opacity in specific regions [S1, S2].
3.  **Densification and Pruning:** The standard GS densification strategy is applied:
    *   Gaussians with low opacity are pruned (reduced) [S1].
    *   New Gaussians are introduced into empty 3D regions [S1].
    *   The binary mask ensures that the distinction between positive and negative Gaussians is maintained during these operations [S1].

### Mathematical Details: Optimal Coefficient and Null Density
The method relies on the properties of the ratio of two Gaussians. The optimal coefficient $c_{opt}$ for a Diff-Gaussian distribution is defined as the minimum of the ratio of the two PDFs [S1]:
$$ c_{opt} := \min_{x} \frac{f_{m_0, \Sigma_0}(x)}{f_{m_1, \Sigma_1}(x)} = \frac{f_{m_0, \Sigma_0}(w)}{f_{m_1, \Sigma_1}(w)} $$
if and only if $w$ satisfies the condition [S1]:
$$ \Sigma_1^{-1}(w - m_1) = \Sigma_0^{-1}(w - m_0) $$
This condition arises from minimizing the quadratic function $q(x) = x^T A x + 2b^T x$, where $A = \Sigma_1^{-1} - \Sigma_0^{-1}$ and $b = \Sigma_0^{-1} m_0 - \Sigma_1^{-1} m_1$, assuming $A$ is positive semi-definite [S1].

For diagonal covariance matrices $\Sigma_0 = \text{diag}(\sigma_1^2, \dots, \sigma_n^2)$ and $\Sigma_1 = \text{diag}(h_1 \sigma_1^2, \dots, h_n \sigma_n^2)$ with $0 < h_i \le 1$, the coordinates $w_i$ of the null density vector (where the ratio is minimized) are given by [S1]:
*   (i) $w_i = \frac{m_{1,i} - h_i m_{0,i}}{1 - h_i}$ if $h_i < 1$
*   (ii) $w_i \in \mathbb{R}$ if $h_i = 1$ and $m_{1,i} = m_{0,i}$

This implies that a null density (zero effective contribution) can be imposed on a $k$-dimensional affine space by aligning means and variances, allowing for precise modeling of complex nonlinear structures [S1].

### Pseudocode
The following pseudocode outlines the initialization and the core distinction in the primitive update step. It assumes standard 3DGS rendering and loss computation are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_neggs(num_total_gaussians, negative_ratio=0.2):
    """
    Initialize the Gaussian primitives for NegGS.
    """
    num_negative = int(num_total_gaussians * negative_ratio)
    num_positive = num_total_gaussians - num_negative
    
    # Initialize positions, scales, rotations, opacities as in standard GS
    # Initialize colors c_i. 
    # For positive Gaussians, c_i > 0.
    # For negative Gaussians, c_i < 0.
    
    gaussians = []
    for i in range(num_positive):
        g = create_gaussian()
        g.color = positive_random_color()
        g.is_negative = False
        gaussians.append(g)
        
    for i in range(num_negative):
        g = create_gaussian()
        g.color = negative_random_color() # Negative color values
        g.is_negative = True
        gaussians.append(g)
        
    return gaussians

def update_gaussians(gaussians, loss, grad):
    """
    Standard GS densification/pruning, but respecting the negative mask.
    """
    # 1. Compute gradients for all attributes including color
    # 2. Prune low opacity Gaussians
    # 3. Clone/Densify high gradient Gaussians
    
    for g in gaussians:
        if g.is_negative:
            # Ensure color remains negative or handle sign flip if optimization drifts
            # Note: Standard GS allows color to be anything, but NegGS relies on
            # the initial negative sign for the "negative Gaussian" identity.
            # The binary mask tracks this identity.
            pass
        else:
            pass
            
    return gaussians
```

### Edge Cases and Pitfalls
*   **Initialization Sensitivity:** The initial number of negative Gaussians is critical. Deviating significantly from the ~20% ratio can reduce optimization efficacy [S1].
*   **Color Sign Drift:** While the mask tracks the identity, optimization might theoretically drive a negative color towards positive. The binary mask ensures the rendering logic treats them as negative contributors regardless of the current color value's sign, but the semantic meaning relies on the initial assignment [S1].
*   **Precision:** The calculation of the Diff-Gaussian ratio involves matrix inversions ($\Sigma^{-1}$). Ensure numerical stability when covariances are ill-conditioned [S1].

## Subtopics

### Diff-Gaussian Distribution
A theoretical construct formed by dividing the PDFs of two Gaussians. It sets NegGS apart by enabling the approximation of non-convex, high-frequency structures (like donuts) that standard Gaussians cannot represent efficiently. Choose this when modeling complex nonlinear geometry is prioritized over simple smooth surfaces. [S2]

### Negative Color Rendering
The practical implementation detail where Gaussians have negative RGB values. This sets NegGS apart from standard GS by allowing destructive interference. Choose this when shadow modeling or high-frequency color transitions are present. [S1, S2]

## Operating characteristics
*   **Time Complexity:** NegGS preserves the original GS rendering speed by leveraging the existing framework and using a binary mask [S1]. However, it alleviates the time complexity increase associated with using many Gaussian components to model complex structures in standard GS [S2].
*   **Memory Complexity:** Similar to standard GS, but potentially fewer total Gaussians are required to achieve comparable or better reconstruction quality for complex scenes, as negative Gaussians allow for more expressive primitives [S1, S2].
*   **Performance:**
    *   On the **Mip-NeRF360** dataset, NegGS-30K achieved an average PSNR of 27.39, SSIM of 0.812, and LPIPS of 0.219, compared to GS-30K's 27.21 PSNR, 0.815 SSIM, and 0.214 LPIPS [S1].
    *   On the **NeRF Synthetic** dataset, NegGS showed slight improvements or parity with GS (e.g., Chair: 35.88 vs 35.82 PSNR) [S1].
*   **Strengths:**
    *   Better modeling of high-frequency elements with rapid color transitions [S1, S2].
    *   Improved representation of shadows [S2].
    *   Ability to model complex nonlinear structures with fewer primitives than standard GS [S1, S2].
*   **Limitations:**
    *   Requires careful tuning of the initial ratio of negative Gaussians [S1].
    *   Theoretical complexity of the Diff-Gaussian ratio may require careful numerical handling [S1].
*   **When to Use:** Use NegGS when the scene contains complex nonlinear structures, high-frequency details, or intricate shadows that standard Gaussian Splatting struggles to represent efficiently without excessive memory usage.
*   **When Not to Use:** For simple, smooth scenes where standard GS is sufficient, the added complexity of managing negative Gaussians may not provide significant benefits.

## Background
Negative Gaussian Splatting was introduced as the first method to extend the simple ellipsoid shapes of Gaussian Splatting to more complex nonlinear structures [S2]. It was developed by researchers publishing under the title "NegGS: Negative Gaussian Splatting" [S1, S2]. The method was evaluated on several benchmarks including NeRF Synthetic, Mip-NeRF360, Tanks and Temples, Deep Blending, and Shiny Blender datasets [S1]. It builds on the standard 3D Gaussian Splatting framework, modifying the representation to allow for negative colors and leveraging the mathematical properties of Gaussian ratios [S1].

## Sources
[S1] NegGS: Negative Gaussian Splatting, https://doi.org/10.1016/j.ins.2025.121912, fetched 2026-10-26
[S2] NegGS: Negative Gaussian Splatting, https://core.ac.uk/download/630026911.pdf, fetched 2026-10-28
