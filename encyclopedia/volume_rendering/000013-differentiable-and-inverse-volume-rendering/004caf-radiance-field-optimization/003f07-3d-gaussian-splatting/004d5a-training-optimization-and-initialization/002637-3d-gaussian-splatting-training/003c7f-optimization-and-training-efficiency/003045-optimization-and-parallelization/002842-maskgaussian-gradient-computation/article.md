# MaskGaussian Gradient Computation

Also called: Gradient of Masks, Mask Gradient Backpropagation

## Overview
MaskGaussian Gradient Computation computes the gradient of a loss function $\mathcal{L}$ with respect to per-Gaussian probabilistic masks $\mathcal{M}_i$ during 3D Gaussian Splatting training [S1]. It optimizes mask parameters that modulate Gaussian opacity, enabling adaptive representation and pruning [S1]. It differs from standard backpropagation by explicitly accounting for the interaction between the mask-modulated Gaussian contribution and the background color contribution, which is modulated by the final transmittance [S1].

## How it works
The method computes the gradient $\frac{\partial \mathcal{L}}{\partial \mathcal{M}_i}$ for each Gaussian $i$ in the sorted list of $N$ Gaussians contributing to a pixel $\mathbf{x}$ [S1].

**Inputs:**
- $\mathcal{L}$: The scalar loss value.
- $\frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})}$: The gradient of the loss with respect to the final rendered pixel color $\mathbf{c}(\mathbf{x})$ (a 3D vector) [S1].
- $\mathbf{c}_i$: The color of the $i$-th Gaussian (a 3D vector) [S1].
- $\alpha_i$: The opacity of the $i$-th Gaussian (scalar) [S1].
- $\mathcal{M}_i$: The mask value for the $i$-th Gaussian (scalar) [S1].
- $\mathbf{c}_{bg}$: The background color (a 3D vector) [S1].
- $N$: The total number of Gaussians contributing to the pixel [S1].

**Data Structures:**
- The Gaussians are sorted by depth [S1].
- Intermediate values $\mathbf{b}_i$ (color rendered from Gaussian $i$ to the last) and $T_i$ (transmittance up to Gaussian $i$) are computed during the forward pass or recomputed [S1].
- $T_{N+1}$: The final transmittance after all $N$ Gaussians [S1].

**Definitions:**
- The total pixel color is split into Gaussian contribution $\mathbf{c}_g(\mathbf{x})$ and background contribution $\mathbf{c}_b(\mathbf{x})$ [S1]:
  $$ \mathbf{c}(\mathbf{x}) = \mathbf{c}_g(\mathbf{x}) + \mathbf{c}_b(\mathbf{x}) $$
- The background contribution is defined as [S1]:
  $$ \mathbf{c}_b(\mathbf{x}) = T_{N+1} \cdot \mathbf{c}_{bg} $$
- The final transmittance is [S1]:
  $$ T_{N+1} = \prod_{j=1}^N (1 - \alpha_j \cdot \mathcal{M}_j) $$
- The intermediate color $\mathbf{b}_i$ is defined recursively from the last Gaussian to the first [S1]:
  $$ \mathbf{b}_i = \mathcal{M}_i \cdot \alpha_i \cdot \mathbf{c}_i + (1 - \mathcal{M}_i \cdot \alpha_i) \cdot \mathbf{b}_{i+1} $$
  where $\mathbf{b}_{N+1} = \mathbf{c}_{bg}$ [S1]. Note: In the standard alpha blending chain, $\mathbf{b}_1 = \mathbf{c}_g(\mathbf{x})$ [S1].
- The transmittance up to Gaussian $i$ is [S1]:
  $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j \cdot \mathcal{M}_j) $$
  with $T_1 = 1$ [S1].

**Steps:**
1. Compute the final transmittance $T_{N+1}$ from the forward pass or by accumulating $(1 - \alpha_j \cdot \mathcal{M}_j)$ for all $j$ [S1].
2. For each Gaussian $i$ from $1$ to $N$:
   a. Compute the gradient of the intermediate color $\mathbf{b}_i$ with respect to the mask $\mathcal{M}_i$ [S1]:
      $$ \frac{\partial \mathbf{b}_i}{\partial \mathcal{M}_i} = \alpha_i \cdot (\mathbf{c}_i - \mathbf{b}_{i+1}) $$
      where $\mathbf{b}_{i+1}$ is the intermediate color from the next Gaussian (for $i=N$, $\mathbf{b}_{N+1}$ is the background color $\mathbf{c}_{bg}$) [S1].
   b. Compute the gradient of the loss with respect to the intermediate color $\mathbf{b}_i$ [S1]:
      $$ \frac{\partial \mathcal{L}}{\partial \mathbf{b}_i} = T_i \frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})} $$
   c. Compute the gradient of the background color contribution with respect to the mask $\mathcal{M}_i$ [S1]:
      $$ \frac{\partial \mathbf{c}_b(\mathbf{x})}{\partial \mathcal{M}_i} = \frac{-\alpha_i \cdot T_{N+1}}{1 - \alpha_i \cdot \mathcal{M}_i} \mathbf{c}_{bg} $$
   d. Combine the gradients to get the total gradient for the mask [S1]:
      $$ \frac{\partial \mathcal{L}}{\partial \mathcal{M}_i} = \frac{\partial \mathcal{L}}{\partial \mathbf{b}_i} \cdot \frac{\partial \mathbf{b}_i}{\partial \mathcal{M}_i} + \frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})} \cdot \frac{\partial \mathbf{c}_b(\mathbf{x})}{\partial \mathcal{M}_i} $$
      Substituting the terms (where $\cdot$ denotes the dot product to yield a scalar) [S1]:
      $$ \frac{\partial \mathcal{L}}{\partial \mathcal{M}_i} = \alpha_i T_i \frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})} \cdot (\mathbf{c}_i - \mathbf{b}_{i+1}) + \frac{-\alpha_i \cdot T_{N+1}}{1 - \alpha_i \cdot \mathcal{M}_i} \frac{\partial \mathcal{L}}{\partial \mathbf{c}(\mathbf{x})} \cdot \mathbf{c}_{bg} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mask_gradients(dL_dc, c_bg, alphas, masks, colors, b_next):
    # dL_dc: gradient of loss w.r.t. pixel color, shape (3,)
    # c_bg: background color, shape (3,)
    # alphas: opacities alpha_i, shape (N,)
    # masks: mask values M_i, shape (N,)
    # colors: Gaussian colors c_i, shape (N, 3)
    # b_next: intermediate colors b_{i+1}, shape (N, 3). b_next[i] stores b_{i+2} (1-based),
    #         i.e., the intermediate color after Gaussian i+1. For i=N-1, b_next[N-1] is b_{N+1} = c_bg.
    # Returns: dL_dM, gradient w.r.t. masks, shape (N,)
    N = len(alphas)
    dL_dM = [0.0] * N

    # Compute final transmittance T_{N+1} = product_{j=0}^{N-1} (1 - alpha_j * M_j)
    T_N_plus_1 = 1.0
    for j in range(N):
        T_N_plus_1 *= (1.0 - alphas[j] * masks[j])

    # Compute transmittance T_i for each i: T_i = product_{j=0}^{i-1} (1 - alpha_j * M_j)
    # T_0 = 1.0 (corresponds to T_1 in 1-based indexing)
    T_i = [1.0] * N
    current_T = 1.0
    for i in range(N):
        T_i[i] = current_T
        current_T *= (1.0 - alphas[i] * masks[i])

    for i in range(N):
        # Gradient of b_i w.r.t. M_i: alpha_i * (c_i - b_{i+1})
        d_bi_d_Mi = [alphas[i] * (colors[i][k] - b_next[i][k]) for k in range(3)]

        # Gradient of L w.r.t. b_i: T_i * dL_dc
        dL_d_bi = [T_i[i] * dL_dc[k] for k in range(3)]

        # Dot product for Gaussian path: dL_d_bi . d_bi_d_Mi
        term1 = sum(dL_d_bi[k] * d_bi_d_Mi[k] for k in range(3))

        # Gradient of c_b(x) w.r.t. M_i: (-alpha_i * T_{N+1} / (1 - alpha_i * M_i)) * c_bg
        denom = 1.0 - alphas[i] * masks[i]
        if denom == 0.0:
            d_c_b_d_Mi = [0.0, 0.0, 0.0] # Guard for division by zero [G]
        else:
            scale = (-alphas[i] * T_N_plus_1) / denom
            d_c_b_d_Mi = [scale * c_bg[k] for k in range(3)]

        # Dot product for background path: dL_dc . d_c_b_d_Mi
        term2 = sum(dL_dc[k] * d_c_b_d_Mi[k] for k in range(3))

        dL_dM[i] = term1 + term2

    return dL_dM
```

**Parameters:**
- $\lambda_m$: A weighting parameter for the mask gradient during training. In the source, $\lambda_m = 0.1$ is used during iterations 19,000 to 20,000 [S1].

**Edge Cases and Pitfalls:**
- Division by zero: The term $\frac{1}{1 - \alpha_i \cdot \mathcal{M}_i}$ becomes undefined if $\alpha_i \cdot \mathcal{M}_i = 1$. A guard or epsilon addition is required [G].
- Numerical stability: Computing $T_{N+1}$ as a product of many terms can lead to underflow. Log-space computation or careful ordering may be required [G].

## Operating characteristics
- Time complexity: Not given in the sources [G].
- Memory complexity: Not given in the sources [G].
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.

## Background
MaskGaussian was introduced in the paper "MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks" [S1]. The method is used in the context of 3D Gaussian Splatting training to improve the efficiency and quality of the Gaussian representation by learning probabilistic masks [S1]. It is compared against LightGaussian, which compresses an already trained 3DGS model, whereas MaskGaussian applies Gaussian pruning during training from scratch [S1].

## Sources
[S1] MaskGaussian: Adaptive 3D Gaussian Representation from Probabilistic Masks, https://doi.org/10.1109/cvpr52734.2025.00072, fetched 2026-10-30
