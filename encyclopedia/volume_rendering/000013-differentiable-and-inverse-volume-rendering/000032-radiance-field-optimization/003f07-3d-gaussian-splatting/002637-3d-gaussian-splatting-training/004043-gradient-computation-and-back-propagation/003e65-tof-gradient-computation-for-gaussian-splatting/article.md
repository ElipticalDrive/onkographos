# ToF Gradient Computation for Gaussian Splatting

Also called: Opacity gradient for ToF image formation, Recursive gradient for quad pixels

## Overview
This method computes the gradient of the loss function with respect to the opacity parameter $\alpha_k$ of the $k$-th Gaussian for Time-of-Flight (ToF) image formation models, specifically for phasor or quad pixel signals [S1]. It differs from standard 3D Gaussian Splatting (3DGS) color rendering by incorporating squared transmittance terms in the recursive accumulation, accounting for the non-linear intensity interactions inherent in ToF physics [S1]. This modification is necessary because the ToF signal formation model does not follow the linear alpha-blending assumption of RGB rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The loss gradient with respect to the rendered ToF signal, $\frac{\partial L}{\partial \mathbf{q}(\mathbf{x})}$ (or $\frac{\partial L}{\partial \mathbf{p}(\mathbf{x})}$ for phasors), where $\mathbf{x}$ is the pixel coordinate.
*   **Input:** The sorted list of $N$ Gaussians intersecting the pixel, ordered back-to-front.
*   **Input:** For each Gaussian $k$, its opacity $\alpha_k$ and its ToF quad contribution $\mathbf{q}_k$.
*   **Output:** The gradient $\frac{\partial L}{\partial \alpha_k}$ for each Gaussian $k$.

**Data Structures:**
*   **Accumulator $\text{acc}_k^q$:** A scalar (or vector, depending on signal dimension) value storing the weighted sum of contributions from Gaussians $k+1$ to $N$.
*   **Transmittance $T_k$:** Defined as $T_k = \prod_{m < k} (1 - \alpha_m)$.

**Recursive Formulas:**
The gradient computation relies on two recursive quantities computed from back to front (from $k=N-1$ down to $0$):

1.  **Accumulation Term ($\text{acc}_k^q$):**
    $$ \text{acc}_k^q = \begin{cases} \alpha_{k+1} \mathbf{q}_{k+1} + (1 - \alpha_{k+1})^2 \text{acc}_{k+1}^q, & k < N-1 \\ 0, & k = N-1 \end{cases} $$
    *Note: The base case is $\text{acc}_{N-1}^q = 0$ because there are no Gaussians after the last one. The recursion index in the source [S1] uses $k < N$ with base case $k=N$, implying 1-based indexing or a specific boundary condition. In 0-based indexing for $N$ Gaussians ($0 \dots N-1$), the loop runs from $N-2$ down to $0$, with $\text{acc}_{N-1}^q = 0$.*

2.  **Gradient Term ($\frac{\partial \mathbf{q}(\mathbf{x})}{\partial \alpha_k}$):**
    $$ \frac{\partial \mathbf{q}(\mathbf{x})}{\partial \alpha_k} = T_k^2 \left( \mathbf{q}_k + 2(\alpha_k - 1)\text{acc}_k^q \right) $$

3.  **Final Loss Gradient:**
    $$ \frac{\partial L}{\partial \alpha_k} = \frac{\partial L}{\partial \mathbf{q}(\mathbf{x})} \cdot \frac{\partial \mathbf{q}(\mathbf{x})}{\partial \alpha_k} $$
    Where $\cdot$ denotes the appropriate dot product or element-wise multiplication depending on whether $\mathbf{q}$ is a scalar intensity or a vector phasor.

**Parameters:**
*   $T_k$: Transmittance up to the $k$-th Gaussian.
*   $\mathbf{q}_k$: The quad contribution of the $k$-th Gaussian.
*   $\alpha_k$: The opacity of the $k$-th Gaussian.

**Pitfalls:**
*   **Squared Terms:** The presence of $(1-\alpha_{k+1})^2$ in the accumulation and $T_k^2$ in the gradient makes the computation less trivial than standard 3DGS [S1].
*   **Indexing:** Care must be taken with the base case of the recursion. The source [S1] defines $\text{acc}_k^q = 0$ for $k=N$. If using 0-based indexing for $N$ Gaussians ($0 \dots N-1$), the accumulator for the last Gaussian ($k=N-1$) is 0.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_tof_opacity_gradient(loss_grad_signal, gaussians):
    """
    Computes gradient of loss w.r.t. opacity alpha_k for ToF quad pixel rendering.
    
    Args:
        loss_grad_signal: Gradient of loss w.r.t. the rendered quad signal q(x). Shape: (D,) or scalar.
        gaussians: List of N Gaussian objects, sorted back-to-front.
                   Each Gaussian has attributes: alpha (scalar), q (signal vector).
    
    Returns:
        grad_alphas: List of gradients dL/d_alpha_k for each Gaussian.
    """
    N = len(gaussians)
    grad_alphas = [0.0] * N
    
    # Initialize accumulator for the last Gaussian (no subsequent Gaussians)
    # acc_q corresponds to acc_k^q in the formula
    acc_q = 0.0
    
    # Compute transmittance T_k for all k
    # T_k = product_{m < k} (1 - alpha_m)
    # We can compute this iteratively or pre-calculate.
    # Let's compute T_k array.
    transmittances = [1.0] * N
    current_T = 1.0
    for k in range(N):
        transmittances[k] = current_T
        current_T *= (1.0 - gaussians[k].alpha)
        
    # Iterate from the last Gaussian (N-1) down to the first (0)
    # The formula for acc_k^q depends on k+1.
    # For k = N-1, acc_{N-1}^q = 0.
    # For k < N-1, acc_k^q = alpha_{k+1} * q_{k+1} + (1 - alpha_{k+1})^2 * acc_{k+1}^q
    
    # We need to process k from N-1 down to 0.
    # However, the gradient for k depends on acc_k^q.
    # The accumulation for k depends on k+1.
    # So we can maintain acc_q as we go backwards.
    
    # Base case: for the last Gaussian (index N-1), acc_q is 0.
    acc_q = 0.0
    
    for k in range(N - 1, -1, -1):
        # 1. Compute gradient for current Gaussian k
        # Formula: d_q/d_alpha_k = T_k^2 * (q_k + 2*(alpha_k - 1)*acc_k^q)
        
        q_k = gaussians[k].q
        alpha_k = gaussians[k].alpha
        T_k = transmittances[k]
        
        # Calculate the derivative of the signal q(x) w.r.t alpha_k
        # Note: The source formula is T_k^2 * (q_k + 2*(alpha_k - 1)*acc_k^q)
        # acc_k^q is the accumulator value BEFORE updating it with k's contribution?
        # Let's re-read the recursion carefully.
        # acc_k^q = alpha_{k+1} q_{k+1} + (1 - alpha_{k+1})^2 acc_{k+1}^q
        # This acc_k^q is used in the gradient formula for k.
        # So, at step k, acc_q holds acc_k^q.
        
        grad_signal_d_alpha_k = (T_k ** 2) * (q_k + 2.0 * (alpha_k - 1.0) * acc_q)
        
        # 2. Compute final loss gradient
        # dL/d_alpha_k = dL/d_q(x) * d_q(x)/d_alpha_k
        # Assuming element-wise multiplication or dot product depending on signal type.
        # If q is a vector, and loss_grad_signal is a vector, this is likely a dot product
        # if the loss is scalar, or element-wise if the loss is per-component.
        # Standard 3DGS uses element-wise for color. Let's assume element-wise for generality
        # or dot product if L is scalar. The source writes partial derivative of vector c(x).
        # If L is scalar, grad_signal is a vector. The result grad_alpha is scalar.
        # So it should be a dot product.
        
        if isinstance(loss_grad_signal, (list, tuple, np.ndarray)) and len(loss_grad_signal) > 1:
            # Dot product for scalar loss
            grad_alphas[k] = np.dot(loss_grad_signal, grad_signal_d_alpha_k)
        else:
            # Element-wise or scalar
            grad_alphas[k] = loss_grad_signal * grad_signal_d_alpha_k
            
        # 3. Update accumulator for the next iteration (which is k-1)
        # We need to compute acc_{k-1}^q using acc_k^q (which is current acc_q)
        # Formula: acc_{k-1}^q = alpha_k * q_k + (1 - alpha_k)^2 * acc_k^q
        # But wait, the loop is going backwards.
        # When we are at k, acc_q is acc_k^q.
        # We need to compute acc_{k-1}^q to use in the next iteration (k-1).
        # So we update acc_q to be acc_{k-1}^q.
        
        if k > 0:
            alpha_next = gaussians[k].alpha # This is alpha_k, which acts as alpha_{k+1} for the previous step?
            # Let's trace indices.
            # We are at k. acc_q is acc_k^q.
            # We want to compute acc_{k-1}^q.
            # Formula: acc_{k-1}^q = alpha_k * q_k + (1 - alpha_k)^2 * acc_k^q
            # Here, alpha_k is the opacity of the Gaussian at index k.
            # In the formula for acc_{k-1}, the term alpha_{k} appears.
            # So yes, we use the current Gaussian's alpha and q.
            
            alpha_k_curr = gaussians[k].alpha
            q_k_curr = gaussians[k].q
            
            acc_q = alpha_k_curr * q_k_curr + ((1.0 - alpha_k_curr) ** 2) * acc_q
            
    return grad_alphas
```

## Operating characteristics
*   **Complexity:** The computation is $O(N)$ per pixel, where $N$ is the number of Gaussians, similar to standard 3DGS. The additional squaring operations add negligible constant overhead.
*   **Memory:** Requires storing the accumulator $\text{acc}_k^q$ for the current pixel, which is $O(1)$ extra memory per pixel during the backward pass.
*   **Strengths:** Correctly models the gradient flow for ToF signals, enabling end-to-end optimization of Gaussian opacities for depth/ToF reconstruction.
*   **Limitations:** The squared terms can lead to numerical instability if opacities are close to 1 or 0, though this is inherent to the ToF model. The method is specific to ToF/quad pixel signals and does not apply to standard RGB rendering.

## Background
This method originates from the paper "Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields" [S1]. It extends the standard 3DGS gradient computation to handle the non-linear intensity interactions in Time-of-Flight imaging, specifically for phasor and quad pixel signals. The standard 3DGS gradient formula $\frac{\partial \mathbf{c}(\mathbf{x})}{\partial \alpha_k} = T_k(\mathbf{c}_k - \alpha \mathbf{c}_k)$ is insufficient for ToF, necessitating the modified recursive accumulators with squared transmittance terms.

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
