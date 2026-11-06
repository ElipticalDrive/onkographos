# MaskGaussian Gradient Computation

Also called: Gradient of Masks, Mask Gradient Backpropagation

## Overview
MaskGaussian Gradient Computation is a method for computing the gradient of a loss function $\mathcal{L}$ with respect to per-Gaussian probabilistic masks $\mathcal{M}_i$ during the training of 3D Gaussian Splatting. It solves the problem of optimizing mask parameters that modulate Gaussian opacity, enabling adaptive Gaussian representation and pruning. It differs from standard 3DGS backpropagation by explicitly accounting for the interaction between the mask-modulated Gaussian contribution and the background color contribution, which is modulated by the final transmittance.

## How it works
The method computes the gradient $\frac{\partial \mathcal{L}}{\partial \mathcal{M}_i}$ for each Gaussian $i$ in the sorted list of $N$ Gaussians contributing to a pixel $\mathbf{x}$.

**Inputs:**
- $\mathcal{L}$: The scalar loss value.
- $\frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})}$: The gradient of the loss with respect to the final rendered pixel color $\mathbf{c}(\mathbf{x})$ (a 3D vector).
- $\mathbf{c}_i$: The color of the $i$-th Gaussian (a 3D vector).
- $\alpha_i$: The opacity of the $i$-th Gaussian (scalar).
- $\mathcal{M}_i$: The mask value for the $i$-th Gaussian (scalar).
- $\mathbf{c}_{bg}$: The background color (a 3D vector).
- $N$: The total number of Gaussians contributing to the pixel.

**Data Structures:**
- The Gaussians are sorted by depth.
- Intermediate values $\mathbf{b}_i$ (color rendered from Gaussian $i$ to the last) and $T_i$ (transmittance up to Gaussian $i$) are computed during the forward pass or recomputed.
- $T_{N+1}$: The final transmittance after all $N$ Gaussians.

**Definitions:**
- The total pixel color is split into Gaussian contribution $\mathbf{c}_g(\mathbf{x})$ and background contribution $\mathbf{c}_b(\mathbf{x})$:
  $$ \mathbf{c}(\mathbf{x}) = \mathbf{c}_g(\mathbf{x}) + \mathbf{c}_b(\mathbf{x}) $$
- The background contribution is defined as:
  $$ \mathbf{c}_b(\mathbf{x}) = T_{N+1} \cdot \mathbf{c}_{bg} $$
- The final transmittance is:
  $$ T_{N+1} = \prod_{j=1}^N (1 - \alpha_j \cdot \mathcal{M}_j) $$
- The intermediate color $\mathbf{b}_i$ is defined recursively from the last Gaussian to the first:
  $$ \mathbf{b}_i = \mathcal{M}_i \cdot \alpha_i \cdot \mathbf{c}_i + (1 - \mathcal{M}_i \cdot \alpha_i) \cdot \mathbf{b}_{i+1} $$
  where $\mathbf{b}_{N+1} = \mathbf{c}_{bg}$ (implicitly, as the background color is the base case for the alpha blending chain if we consider the background as the "last" element, but formally $\mathbf{c}_b$ is separated). Note: In the standard alpha blending chain, $\mathbf{b}_1 = \mathbf{c}_g(\mathbf{x})$.
- The transmittance up to Gaussian $i$ is:
  $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j \cdot \mathcal{M}_j) $$
  with $T_1 = 1$.

**Steps:**
1. Compute the final transmittance $T_{N+1}$ from the forward pass or by accumulating $(1 - \alpha_j \cdot \mathcal{M}_j)$ for all $j$.
2. For each Gaussian $i$ from $1$ to $N$:
   a. Compute the gradient of the intermediate color $\mathbf{b}_i$ with respect to the mask $\mathcal{M}_i$:
      $$ \frac{\partial \mathbf{b}_i}{\partial \mathcal{M}_i} = \alpha_i \cdot (\mathbf{c}_i - \mathbf{b}_{i+1}) $$
      where $\mathbf{b}_{i+1}$ is the intermediate color from the next Gaussian (for $i=N$, $\mathbf{b}_{N+1}$ is effectively the background color $\mathbf{c}_{bg}$ in the context of the Gaussian chain, but strictly $\mathbf{b}_{N+1}$ in the alpha blending definition is the color passed through, which is $\mathbf{c}_{bg}$ if no more Gaussians exist).
   b. Compute the gradient of the loss with respect to the intermediate color $\mathbf{b}_i$:
      $$ \frac{\partial \mathcal{L}}{\partial \mathbf{b}_i} = T_i \frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})} $$
   c. Compute the gradient of the background color contribution with respect to the mask $\mathcal{M}_i$:
      $$ \frac{\partial \mathbf{c}_b(\mathbf{x})}{\partial \mathcal{M}_i} = \frac{-\alpha_i \cdot T_{N+1}}{1 - \alpha_i \cdot \mathcal{M}_i} \mathbf{c}_{bg} $$
      Note: If $1 - \alpha_i \cdot \mathcal{M}_i$ is zero, this term is undefined. In practice, a small epsilon should be added to the denominator to avoid division by zero [G].
   d. Combine the gradients to get the total gradient for the mask:
      $$ \frac{\partial \mathcal{L}}{\partial \mathcal{M}_i} = \frac{\partial \mathcal{L}}{\partial \mathbf{b}_i} \frac{\partial \mathbf{b}_i}{\partial \mathcal{M}_i} + \frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})} \frac{\partial \mathbf{c}_b(\mathbf{x})}{\partial \mathcal{M}_i} $$
      Substituting the terms:
      $$ \frac{\partial \mathcal{L}}{\partial \mathcal{M}_i} = \alpha_i T_i \frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})} (\mathbf{c}_i - \mathbf{b}_{i+1}) + \frac{-\alpha_i \cdot T_{N+1}}{1 - \alpha_i \cdot \mathcal{M}_i} \frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})} \mathbf{c}_{bg} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mask_gradients(
    dL_dc: np.ndarray,  # Gradient of loss w.r.t. pixel color, shape (3,)
    c_bg: np.ndarray,   # Background color, shape (3,)
    alphas: np.ndarray, # Opacities of N Gaussians, shape (N,)
    masks: np.ndarray,  # Masks of N Gaussians, shape (N,)
    colors: np.ndarray, # Colors of N Gaussians, shape (N, 3)
    b_next: np.ndarray  # Intermediate color b_{i+1} for each i, shape (N+1, 3). b_next[N] is c_bg.
) -> np.ndarray:
    """
    Computes the gradient of the loss with respect to the masks for all N Gaussians.
    
    Args:
        dL_dc: Gradient of loss w.r.t. final pixel color c(x).
        c_bg: Background color.
        alphas: Array of opacities alpha_i for i in 0..N-1.
        masks: Array of mask values M_i for i in 0..N-1.
        colors: Array of Gaussian colors c_i for i in 0..N-1.
        b_next: Array of intermediate colors b_{i+1}. b_next[i] corresponds to b_{i+1} for Gaussian i.
                b_next[N] should be c_bg.
                
    Returns:
        dL_dM: Gradient of loss w.r.t. masks, shape (N,).
    """
    N = len(alphas)
    dL_dM = np.zeros(N)
    
    # Compute final transmittance T_{N+1}
    # T_{N+1} = product_{j=0}^{N-1} (1 - alpha_j * M_j)
    transmittance_factors = 1.0 - alphas * masks
    T_N_plus_1 = np.prod(transmittance_factors)
    
    # Compute transmittance T_i for each i
    # T_i = product_{j=0}^{i-1} (1 - alpha_j * M_j)
    # T_0 = 1 (for i=0, which is the first Gaussian)
    # We can compute this cumulatively
    T_i = np.ones(N + 1)
    for i in range(N):
        T_i[i+1] = T_i[i] * transmittance_factors[i]
    # T_i[i] is the transmittance up to Gaussian i (before Gaussian i)
    
    for i in range(N):
        # Gradient of b_i w.r.t. M_i: alpha_i * (c_i - b_{i+1})
        # b_next[i] is b_{i+1}
        d_bi_d_Mi = alphas[i] * (colors[i] - b_next[i])
        
        # Gradient of L w.r.t. b_i: T_i * dL_dc
        # T_i[i] is the transmittance up to Gaussian i
        dL_d_bi = T_i[i] * dL_dc
        
        # Gradient of L w.r.t. M_i from the Gaussian path
        term1 = dL_d_bi * d_bi_d_Mi
        
        # Gradient of c_b(x) w.r.t. M_i
        # d_c_b_d_Mi = (-alpha_i * T_{N+1} / (1 - alpha_i * M_i)) * c_bg
        denom = 1.0 - alphas[i] * masks[i]
        # Guard against division by zero
        if np.isclose(denom, 0.0):
            d_c_b_d_Mi = np.zeros(3) # Or handle as undefined/singular
        else:
            d_c_b_d_Mi = (-alphas[i] * T_N_plus_1 / denom) * c_bg
            
        # Gradient of L w.r.t. M_i from the background path
        term2 = dL_dc * d_c_b_d_Mi
        
        # Total gradient
        dL_dM[i] = np.sum(term1 + term2)
        
    return dL_dM
```

**Parameters:**
- $\lambda_m$: A weighting parameter for the mask gradient during training. In the source, $\lambda_m = 0.1$ is used during iterations 19,000 to 20,000 [S1].

**Edge Cases and Pitfalls:**
- Division by zero: The term $\frac{1}{1 - \alpha_i \cdot \mathcal{M}_i}$ can become undefined if $\alpha_i \cdot \mathcal{M}_i = 1$. A small epsilon should be added to the denominator [G].
- Numerical stability: Computing $T_{N+1}$ as a product of many terms can lead to underflow. Log-space computation or careful ordering may be required [G].
- Degenerate input: If a Gaussian has zero opacity or zero mask, the gradient terms simplify.

## Operating characteristics
- Time Complexity: $O(N)$ per pixel, where $N$ is the number of Gaussians contributing to the pixel. This is comparable to the forward pass.
- Memory Complexity: $O(N)$ to store intermediate values like $T_i$ and $\mathbf{b}_{i+1}$ if not cached from the forward pass.
- Strengths: Enables end-to-end optimization of Gaussian masks, allowing for adaptive pruning and representation.
- Limitations: Requires careful handling of the background interaction term, which adds computational overhead compared to standard 3DGS backpropagation.

## Background
MaskGaussian was introduced in the paper "MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks" [S1]. The method is used in the context of 3D Gaussian Splatting training to improve the efficiency and quality of the Gaussian representation by learning probabilistic masks. It is compared against LightGaussian, which compresses an already trained 3DGS model, whereas MaskGaussian applies Gaussian pruning during training from scratch [S1].

## Sources
[S1] MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks, https://doi.org/10.1109/cvpr52734.2025.00072, fetched 2026-10-30
