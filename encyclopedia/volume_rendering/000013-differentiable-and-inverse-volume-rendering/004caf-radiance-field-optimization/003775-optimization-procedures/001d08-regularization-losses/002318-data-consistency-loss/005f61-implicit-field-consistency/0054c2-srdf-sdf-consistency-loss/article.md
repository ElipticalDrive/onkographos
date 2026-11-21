# SRDF-SDF Consistency Loss


## Overview
The SRDF-SDF Consistency Loss enforces sign consistency between a predicted Signed Ray Distance Function (SRDF) and a Signed Distance Function (SDF), ensuring both indicate the same spatial occupancy (positive outside, negative inside) [S1]. It addresses the issue where separate network branches predict SRDF and SDF without guaranteeing aligned signs [S1]. Unlike the [Density Consistency Loss](17213-density-consistency-loss.md), which focuses on density field alignment, this method operates directly on the geometric distance fields to penalize sign discrepancies [S1].

## How it works
**Inputs and Outputs:**
- Inputs: Predicted SRDF values $\tilde{d}_{\Omega}$ and SDF values $d_{\Omega}$ for a set of sampled points $\mathbf{p}$ along rays $\mathbf{r}$.
- Output: A scalar loss value $\mathcal{L}_{con}$.

**Data Structures:**
- $\tilde{d}_{\Omega}$: Tensor of SRDF values.
- $d_{\Omega}$: Tensor of SDF values.
- $M_{con}$: Binary mask tensor where $M_{con} = 1$ if $\tilde{d}_{\Omega} \cdot d_{\Omega} < 0$, else $0$.
- $N_r$: Count of points where $M_{con} = 1$.

**Parameters:**
- $k$: Hyperparameter controlling the slope of the sigmoid function used to approximate the sign function [S1]. Typical values are not specified in the sources.

**Steps:**
1. Identify points with opposite signs between SRDF and SDF using the indicator function $M_{con} = [\tilde{d}_{\Omega} \cdot d_{\Omega} < 0]$ [S1].
2. Count the number of such points $N_r$ in the minibatch [S1].
3. Compute the differentiable sign approximation $\varsigma(d) = \text{Sigmoid}(k \cdot d)$ for both $\tilde{d}_{\Omega}$ and $d_{\Omega}$ [S1].
4. Calculate the $\ell_2$ norm of the difference between the approximated signs [S1].
5. Compute the loss:
   $$ \mathcal{L}_{con} = \frac{1}{N_r}\sum_{\mathbf{p}, \mathbf{r}} M_{con}||\varsigma (\tilde{d}_{\Omega })-\varsigma (d_{\Omega })||_2 $$
   [S1]

**Derivative:**
The derivative with respect to the SDF $d_{\Omega}$ is:
$$ \frac{\partial \mathcal{L}_{con}}{\partial d_{\Omega }} = -\frac{2}{N_r}M_{con}||\varsigma (\tilde{d}_{\Omega })-\varsigma (d_{\Omega })||_1 \cdot \frac{\partial \varsigma (d_{\Omega })}{\partial d_{\Omega }} $$
[S1]

**Pitfalls:**
- The sign function is non-differentiable, so the sigmoid approximation is necessary for back-propagation [S1].
- $N_r$ must be non-zero; if no points have opposite signs, the loss is undefined or zero depending on implementation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def srdf_sdf_consistency_loss(tilde_d, d, k):
    """
    Compute SRDF-SDF consistency loss.
    
    Args:
        tilde_d: Tensor of SRDF values.
        d: Tensor of SDF values.
        k: Hyperparameter for sigmoid slope.
        
    Returns:
        Loss scalar.
    """
    # Check for opposite signs
    opposite_sign_mask = (tilde_d * d) < 0
    
    # Count points with opposite signs
    N_r = opposite_sign_mask.sum()
    
    # Avoid division by zero
    if N_r == 0:
        return torch.tensor(0.0, device=tilde_d.device)
    
    # Approximate sign using sigmoid
    # Sigmoid(k * d) approximates sign(d)
    sig_tilde = torch.sigmoid(k * tilde_d)
    sig_d = torch.sigmoid(k * d)
    
    # Compute L2 norm of difference in sign approximations
    # Note: The formula uses ||...||_2 inside the sum, but the derivative uses ||...||_1.
    # The source formula (6) shows ||...||_2. We follow the source formula for the loss value.
    diff = sig_tilde - sig_d
    loss_per_point = torch.norm(diff, p=2) # Element-wise L2 norm for scalars is absolute value, but source implies vector norm if multi-dimensional. 
    # However, d and tilde_d are scalars per point. ||a - b||_2 for scalars is |a - b|.
    # The source notation ||...||_2 likely refers to the Euclidean norm of the vector of differences if summed, 
    # but the formula has sum_{p,r} outside. Let's assume element-wise absolute difference for scalar fields.
    # Re-reading source: "||\varsigma (\tilde{d}_{\Omega })-\varsigma (d_{\Omega })||_2". 
    # If these are scalars, L2 norm is absolute value.
    
    # Mask the loss
    masked_loss = opposite_sign_mask.float() * torch.abs(diff)
    
    # Sum and normalize
    loss = masked_loss.sum() / N_r
    
    return loss
```

## Subtopics

### SRDF Consistency Energy
This variant enforces consistency across multiple cameras by minimizing an energy function $E$ based on SRDF values [S2]. It computes the energy at sample locations along viewing rays and back-propagates gradients to update depth maps [S2]. The SRDF for camera $j$ at point $X$ is defined as $SRDF_j(X) = D_j(X) - Z_j(X)$, where $D_j(X)$ is the depth in the depth map and $Z_j(X)$ is the distance from $X$ to camera $j$ [S2]. The energy is:
$$ E(\{X\}, D, I) = \prod_{j} \exp \left( -\frac{SRDF_j(X)^2}{\sigma_d} \right) + \Gamma_{SRDF} $$
[S2]
Parameters: $\sigma_d$ controls the decay rate of probabilities as SRDF deviates from zero; $\Gamma_{SRDF}$ prevents the product from canceling out due to occlusions, interpreted as the probability of occlusion [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of sampled points $N_r$ [S1].
- **Memory Complexity:** Linear with respect to the number of sampled points to store $\tilde{d}_{\Omega}$, $d_{\Omega}$, and masks [S1].
- **Strengths:** Provides a differentiable approximation of sign consistency, allowing gradient-based optimization of geometric fields [S1]. It adjusts penalties based on the extent of sign discrepancy [S1].
- **Limitations:** Relies on the sigmoid approximation, which may introduce bias depending on $k$ [S1]. Requires $N_r > 0$ for non-zero loss [G].
- **When to use:** When SRDF and SDF are predicted by separate branches and sign alignment is critical [S1].

## Background
The SRDF-SDF Consistency Loss was introduced in the context of Ray-Distance Volume Rendering for Neural Scene Reconstruction [S1]. It addresses the lack of guaranteed sign consistency when SDF and SRDF are predicted from separate network branches [S1]. The method uses a sigmoid function to approximate the non-differentiable sign function for back-propagation [S1].

## Sources
[S1] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
[S2] Multi-View Reconstruction Using Signed Ray Distance Functions (SRDF), https://doi.org/10.1109/cvpr52729.2023.01602, fetched 2026-10-26
