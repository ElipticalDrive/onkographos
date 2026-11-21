# Softmax Depth Gradient Derivation

Also called: Softmax Depth Backward Pass, Softmax Depth Gradient Accumulator, Softmax-Scaling Depth, softmax depth

## Overview
Softmax Depth Gradient Derivation computes the gradients for a depth rendering formulation that uses softmax scaling to bias accumulation toward high-weight Gaussians while allowing gradients to propagate to off-mode Gaussians [S1]. It solves the problem of mode-selection depth, which restricts gradients to the single highest-weight Gaussian and fails when closer Gaussians are translucent [S1]. Unlike mode-selection, this method approximates mode depth while maintaining gradient flow to all contributing Gaussians, modulated by a temperature parameter $\beta$ [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $N$: Number of Gaussians along the ray.
    *   $d_i$: Depth of the $i$-th Gaussian.
    *   $\alpha_i$: Opacity of the $i$-th Gaussian.
    *   $\beta$: Softmax temperature parameter.
    *   $\frac{\partial L}{\partial d^{softmax}}$: Gradient of the loss with respect to the rendered softmax depth.
*   **Outputs:**
    *   $\frac{\partial L}{\partial \alpha_k}$: Gradient of the loss with respect to the opacity of the $k$-th Gaussian.
    *   $\frac{\partial L}{\partial d_k}$: Gradient of the loss with respect to the depth of the $k$-th Gaussian.

**Data Structures**
*   **Accumulators:** Two arrays $A^l$ and $A^r$ of size $N+1$, initialized to 0. These store intermediate sums to avoid $O(N^2)$ complexity in the backward pass [S1].

**Definitions**
Let the Gaussians be ordered from closest ($k=1$) to farthest ($k=N$) [S1].
Define the weight $w_k$ and transmittance $T_k$ as:
$$
w_k = T_k \alpha_k
$$
$$
T_k = \prod_{i=1}^{k-1} (1 - \alpha_i)^{1/2}
$$
The rendered softmax depth $d^{softmax}_{x,y}$ is defined as [S1]:
$$
d^{softmax}_{x,y} = \log \left( \frac{\sum_{i=1}^N w_i e^{\beta w_i} d_i}{\sum_{i=1}^N w_i e^{\beta w_i}} \right)
$$
Let $S = \sum_{i=1}^N w_i e^{\beta w_i}$. The forward pass computes $S$ and the numerator sum, which are passed to the backward pass [S1].

**Gradient Derivation**
The gradient with respect to the depth $d_k$ is derived directly from the log-sum-exp structure [S1]:
$$
\frac{\partial d^{softmax}_{x,y}}{\partial d_k} = \frac{w_k e^{\beta w_k}}{S}
$$
The gradient with respect to the opacity $\alpha_k$ is more complex. Let $U_k = (1 + \beta w_k) e^{\beta w_k}$ [S1]. The derivative is:
$$
\frac{\partial d^{softmax}_{x,y}}{\partial \alpha_k} = T_k U_k d_k - \frac{1}{S} \sum_{j=k+1}^N (1 - \alpha_k) U_j d_j - \frac{1}{S} \left( T_k U_k - \sum_{j=k+1}^N (1 - \alpha_k) U_j \right) \frac{\sum_{i=1}^N w_i e^{\beta w_i} d_i}{S}
$$
To compute this efficiently, we use accumulators $A^l_k$ and $A^r_k$ [S1].

**Algorithm Steps**
1.  **Forward Pass:**
    *   Compute $T_k$ for all $k$. Note $T_1 = 1$.
    *   Compute $w_k = T_k \alpha_k$ for all $k$.
    *   Compute $S = \sum_{i=1}^N w_i e^{\beta w_i}$.
    *   Compute $d^{softmax}_{x,y} = \log \left( \frac{\sum_{i=1}^N w_i e^{\beta w_i} d_i}{S} \right)$.
    *   Store $S$, $T_k$, $w_k$, and $\alpha_k$ for the backward pass [S1].

2.  **Backward Pass (Accumulator Initialization):**
    *   Initialize $A^l_0 = 0$ and $A^r_0 = 0$ [S1].

3.  **Backward Pass (Accumulator Update):**
    *   Iterate $k$ from 1 to $N-1$ (or up to $N$ depending on indexing convention, but the recurrence uses $k-1$ to compute $k$):
        $$
        A^l_k = \alpha_{k-1} U_{k-1} d_{k-1} + (1 - \alpha_{k-1}) A^l_{k-1}
        $$
        $$
        A^r_k = \alpha_{k-1} U_{k-1} + (1 - \alpha_{k-1}) A^r_{k-1}
        $$
    *   Here, $U_{k-1} = (1 + \beta w_{k-1}) e^{\beta w_{k-1}}$ [S1].

4.  **Gradient Computation:**
    *   For each $k$, the gradient $\frac{\partial d^{softmax}_{x,y}}{\partial \alpha_k}$ is computed using the accumulated values. The specific formula in the source uses $A^l_k$ and $A^r_k$ to represent the tail sums. Based on the source equation (4) and the accumulator definitions:
        $$
        \frac{\partial d^{softmax}_{x,y}}{\partial \alpha_k} = T_k U_k d_k - \frac{A^l_k}{S} - \frac{A^r_k}{S} \left( \frac{\sum_{i=1}^N w_i e^{\beta w_i} d_i}{S} \right)
        $$
        *Note: The source equation (4) shows a subtraction of terms involving sums from $j=k+1$ to $N$. The accumulators $A^l_k$ and $A^r_k$ defined in the source are updated using index $k-1$ to produce values for index $k$. Thus, $A^l_k$ corresponds to the sum up to $k-1$ scaled by transmittance. The exact mapping requires careful alignment of indices. The source states: "Let $A^l_k, A^r_k$ be the accumulators pertaining to ... for expressions (3) and (4) respectively." and gives the recurrence for $A^l_k$ using $k-1$. This implies $A^l_k$ holds the accumulated value for the term associated with $\alpha_k$'s dependency on previous alphas.*
    *   The gradient for depth is:
        $$
        \frac{\partial d^{softmax}_{x,y}}{\partial d_k} = \frac{w_k e^{\beta w_k}}{S}
        $$

**Parameters**
*   $\beta$: Softmax temperature parameter. Typical value: 5 [S1].
*   Limits: $\lim_{\beta \to 0} d^{softmax}_{x,y} = \log(d^{alpha}_{x,y})$ and $\lim_{\beta \to \infty} d^{softmax}_{x,y} = \log(d^{mode}_{x,y})$ [S1].

**Pitfalls**
*   Computing the derivative w.r.t $\alpha$ directly for all terms results in $O(N^2)$ complexity. The accumulator strategy is required to reduce this to $O(N)$ [S1].
*   Numerical stability: The exponential $e^{\beta w_i}$ can overflow if $\beta w_i$ is large. Standard log-sum-exp tricks may be needed if $\beta$ is large, though the source does not explicitly detail this implementation guard [G].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel for both forward and backward passes, due to the accumulator strategy [S1].
*   **Memory Complexity:** $O(N)$ to store weights, transmittance, and accumulators [S1].
*   **Strengths:** Allows gradients to propagate to Gaussians off the mode, which is beneficial when closer Gaussians are translucent [S1]. Approximates mode depth while avoiding the hard selection of mode-selection [S1].
*   **Limitations:** Requires additional computation for the softmax scaling compared to standard alpha-blending depth [S1].
*   **When to use:** When depth correlation loss is used in training 3D Gaussian Splatting to regularize geometry, especially in sparse view settings where floater pruning is needed [S1].

## Background
*   **Origin:** Introduced in "SparseGS: Sparse View Synthesis Using 3D Gaussian Splatting" [S1].
*   **History:** Developed to address limitations of mode-selection depth in 3DGS training, specifically the issue where gradients do not flow to Gaussians that are not the mode, leading to inaccurate depth estimation when translucent objects are present [S1].
*   **Usage:** Used in the rasterizer for depth rendering in 3DGS training pipelines, particularly in conjunction with depth correlation loss and view sampling for Unconstrained View Reconstruction (UVR) [S1].

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
