# Adaptive-Scale Color Offset


## Overview
Adaptive-Scale Color Offset (ACO) is a training-stage module for multi-scale 3D Gaussian Splatting that corrects scale-dependent color mismatch by applying a learnable, scale-specific RGB offset to the shared base color of each Gaussian [S1]. It sets itself apart from sibling methods like [Alias-free 4D Gaussian Splatting](9872-alias-free-4d-gaussian-splatting.md) by modifying only the Spherical Harmonics (SH) DC coefficient to adapt color per scale, while keeping Gaussian geometry and higher-order view-dependent SH coefficients unchanged [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $N$: Number of Gaussians.
    *   $S$: Number of rendering scales.
    *   $B$: Number of SH basis functions (typically $B=9$ for degree 1, $B=27$ for degree 2).
    *   $f_{i,b} \in \mathbb{R}^3$: The original SH coefficients for Gaussian $i$ and basis $b$.
    *   $Y_b(d)$: The value of the $b$-th SH basis function evaluated at view direction $d$.
    *   $s_{target}$: The index of the target rendering scale during inference ($0 \le s_{target} < S$).
*   **Outputs:**
    *   $c_{i, s_{target}}(d)$: The corrected color of Gaussian $i$ at scale $s_{target}$ for view direction $d$.

**Data Structures**
*   **SH Coefficients ($f$):** A tensor of shape $(N, B, 3)$. $f_{i,b}$ stores the RGB coefficients for Gaussian $i$ and basis $b$.
*   **ACO Offsets ($\Delta$):** A learnable parameter tensor of shape $(N, S, 3)$.
    *   $\Delta_{i,s} \in \mathbb{R}^3$ is the RGB offset for Gaussian $i$ at scale $s$.
    *   Initialized to zero: $\Delta_{i,s} = \mathbf{0}$ for all $i, s$ [S1].

**Numbered Steps**

1.  **Initialization:** Initialize the ACO offset matrix $\Delta \in \mathbb{R}^{N \times S \times 3}$ with zeros [S1].
2.  **Training Loop:**
    a. Select a training scale $s$ from the set of $S$ scales.
    b. Render the scene at scale $s$ using the standard 3DGS splatting algorithm.
    c. Compute the reconstruction loss $\mathcal{L}$ between the rendered image and the ground truth image [S1].
    d. Back-propagate $\mathcal{L}$ to update the SH coefficients $f$ and the specific ACO offsets $\Delta_{i,s}$ for the selected scale $s$. Note that $\Delta_{i,s'}$ for $s' \neq s$ are not updated in this step [S1].
3.  **Inference/Rendering:**
    a. Given a target rendering scale $s_{target}$.
    b. For each Gaussian $i$ and view direction $d$:
        i. Retrieve the original DC coefficient $f_{i,0}$.
        ii. Retrieve the scale-specific offset $\Delta_{i, s_{target}}$.
        iii. Compute the adapted DC coefficient: $f_{i,0}^{s_{target}} = f_{i,0} + \Delta_{i, s_{target}}$ [S1].
        iv. Compute the color using the adapted DC coefficient and the shared higher-order coefficients:
        $$c_{i, s_{target}}(d) = Y_0(d) f_{i,0}^{s_{target}} + \sum_{b=1}^{B-1} Y_b(d) f_{i,b}$$
        v. Use $c_{i, s_{target}}(d)$ for splatting [S1].

**Formulas**
The base color representation for Gaussian $i$ along view direction $d$ is given by the SH expansion [S1]:
$$c_i(d) = \sum_{b=0}^{B-1} Y_b(d) f_{i,b}$$
where $Y_b(d)$ is the $b$-th SH basis function and $f_{i,b}$ is the corresponding SH coefficient [S1].

The scale-adapted color at scale $s$ is defined as [S1]:
$$c_{i,s}(d) = Y_0(d) f_{i,0}^s + \sum_{b=1}^{B-1} Y_b(d) f_{i,b}$$
where the adapted DC coefficient is $f_{i,0}^s = f_{i,0} + \Delta_{i,s}$ [S1]. Here, $\Delta_{i,s} \in \mathbb{R}^3$ is the learnable DC color offset for Gaussian $i$ at scale $s$ [S1].

**Parameters**
*   **Number of Scales ($S$):** Not given in the sources.
*   **SH Degree:** Not given in the sources, but $B$ is the number of basis functions.
*   **Initialization:** $\Delta_{i,s} = 0$ [S1].

**Edge Cases and Pitfalls**
*   **Geometry Independence:** ACO modifies only the DC color term. It leaves the Gaussian geometry (position, covariance/scale) unchanged [S1].
*   **Shared Higher-Order Coefficients:** The higher-order SH coefficients ($b > 0$) remain shared across all scales and are not modified by the offset [S1].
*   **Optimization Scope:** The DC offsets are optimized through the backbone reconstruction loss alone. The loss back-propagates to the corresponding offset $\Delta_{i,s}$ only at the rendering scale $s$ being trained [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class AdaptiveScaleColorOffset(nn.Module):
    """
    ACO Module for Multi-Scale 3D Gaussian Splatting.
    
    Attributes:
        num_gaussians (int): N
        num_scales (int): S
        num_sh_bases (int): B
        sh_coeffs (torch.Tensor): Shape (N, B, 3). Original SH coefficients.
        offsets (torch.Tensor): Shape (N, S, 3). Learnable DC offsets.
    """
    def __init__(self, num_gaussians, num_scales, num_sh_bases):
        super().__init__()
        self.num_gaussians = num_gaussians
        self.num_scales = num_scales
        self.num_sh_bases = num_sh_bases
        
        # SH coefficients are typically managed by the main 3DGS optimizer,
        # but for this module's context, we assume they are accessible.
        # Here we define the learnable ACO parameters.
        self.offsets = nn.Parameter(torch.zeros(num_gaussians, num_scales, 3))
        
    def forward(self, sh_coeffs, view_directions, target_scale):
        """
        Compute scale-adapted colors.
        
        Args:
            sh_coeffs (torch.Tensor): Shape (N, B, 3). Original SH coefficients.
            view_directions (torch.Tensor): Shape (N, 3). View directions for each Gaussian.
            target_scale (int): The index of the rendering scale (0 to S-1).
            
        Returns:
            torch.Tensor: Shape (N, 3). Corrected colors.
        """
        # Extract DC coefficient (b=0) and higher-order coefficients (b=1 to B-1)
        dc_coeff = sh_coeffs[:, 0, :]  # Shape (N, 3)
        higher_order_coeffs = sh_coeffs[:, 1:, :]  # Shape (N, B-1, 3)
        
        # Retrieve the scale-specific offset
        # offsets shape: (N, S, 3)
        scale_offset = self.offsets[:, target_scale, :]  # Shape (N, 3)
        
        # Compute adapted DC coefficient
        adapted_dc = dc_coeff + scale_offset  # Shape (N, 3)
        
        # Compute SH basis values for the view directions
        # Y_b(d) for b=0 is constant (sqrt(1/4pi) in standard normalization, 
        # but often absorbed into coefficients. We assume standard SH evaluation).
        # For implementation, we need a function to evaluate SH bases.
        # Assuming a helper function `evaluate_sh_bases` exists or is implemented.
        # Y shape: (N, B)
        Y = evaluate_sh_bases(view_directions, self.num_sh_bases)
        
        # Y_0 is the first column
        Y_0 = Y[:, 0:1]  # Shape (N, 1)
        Y_higher = Y[:, 1:]  # Shape (N, B-1)
        
        # Compute color contribution from adapted DC
        # Y_0 is (N, 1), adapted_dc is (N, 3). Expand Y_0 to (N, 1, 3) for broadcasting
        color_dc = Y_0.unsqueeze(2) * adapted_dc.unsqueeze(1)  # Shape (N, 1, 3)
        
        # Compute color contribution from higher-order SH
        # Y_higher is (N, B-1), higher_order_coeffs is (N, B-1, 3)
        # We need to sum over the B-1 dimension
        # Y_higher.unsqueeze(2) is (N, B-1, 1)
        # higher_order_coeffs is (N, B-1, 3)
        color_higher = torch.sum(Y_higher.unsqueeze(2) * higher_order_coeffs, dim=1)  # Shape (N, 3)
        
        # Combine
        color = color_dc.squeeze(1) + color_higher  # Shape (N, 3)
        
        return color

def evaluate_sh_bases(view_directions, num_bases):
    """
    Evaluates SH basis functions for given view directions.
    This is a placeholder for the actual SH basis evaluation logic.
    In a real implementation, this would use the standard SH polynomial formulas.
    """
    # Standard SH basis evaluation logic would go here.
    # For the purpose of this article, we assume this function is available.
    # It returns a tensor of shape (N, num_bases).
    pass
```

## Subtopics

### Error-Guided Gaussian Reallocation (EGR)
A companion module to ACO that redistributes low-contribution Gaussians toward high-error regions to address primitive allocation imbalance. It uses contribution and error-aware selection under a fixed Gaussian budget. [S1]

### Scale-Adaptive Gaussian Splatting
The parent method that ACO extends. It involves rendering at multiple scales to handle aliasing and detail. [Parent]

### Alias-free 4D Gaussian Splatting
A sibling method that addresses aliasing in dynamic scenes, differing from ACO by not using scale-specific color offsets but rather other anti-aliasing techniques. [Sibling]

## Operating characteristics
*   **Time Complexity:** Rendering time is not significantly increased by ACO, as it only adds a vector addition to the DC coefficient per Gaussian. The training time increases slightly due to the optimization of the offset parameters $\Delta$ [S1].
*   **Memory Complexity:** ACO adds $N \times S \times 3$ parameters to the model, where $N$ is the number of Gaussians and $S$ is the number of scales. This is a small overhead compared to the SH coefficients [S1].
*   **Strengths:** Corrects color mismatch at coarse rendering scales without altering Gaussian geometry. It provides scale-dependent color adaptation [S1].
*   **Limitations:** Only corrects color, not geometry or view-dependent shading (higher-order SH). It requires training at multiple scales [S1].
*   **When to use:** When rendering multi-scale 3DGS and observing color inaccuracies at coarse scales. [S1]
*   **When not to use:** When geometry accuracy is the primary concern, or when single-scale rendering is sufficient. [S1]

## Background
ACO was proposed in the paper "Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting" published in *Mathematics* in 2026 [S1]. It addresses the issue where coarse-scale error regions remain underrepresented in multi-scale 3DGS, leading to color mismatch because the same color representation is shared across rendering scales [S1]. The method is part of the ACO-EGR framework, which also includes Error-Guided Gaussian Reallocation (EGR) [S1].

## Sources
[S1] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
