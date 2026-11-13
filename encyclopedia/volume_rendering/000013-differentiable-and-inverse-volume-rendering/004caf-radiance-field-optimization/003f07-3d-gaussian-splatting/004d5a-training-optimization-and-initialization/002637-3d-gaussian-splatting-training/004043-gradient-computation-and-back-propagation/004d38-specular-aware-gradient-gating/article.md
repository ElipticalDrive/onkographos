# Specular-Aware Gradient Gating

Also called: Partial stop-gradient mechanism for transmission

## Overview
Specular-Aware Gradient Gating is a training optimization technique for 3D Gaussian Splatting that suppresses specular artifacts in transparent regions while preserving transmission details [S1, unofficial]. It differs from standard back-propagation by modulating the gradients of the transmission color component using a local variance-based gating factor, ensuring the forward pass remains visually identical to the original model [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: Original transmission color $\mathbf{C}_{\text{trans}}$ at a pixel or Gaussian contribution, local neighborhood of transmission colors for variance calculation.
- Output: Gated transmission color $\tilde{\mathbf{C}}_{\text{trans}}$ used for final composition (forward pass) and modified gradients for back-propagation.

**Data Structures**
- Transmission color $\mathbf{C}_{\text{trans}}$: Vector of color values (e.g., RGB) for the transmission component.
- Gating factor $g$: Scalar value in $[0, 1]$ derived from local variance.

**Parameters**
- Gating strength hyperparameter $k$: Controls the sensitivity of the gating factor to variance. Typical value: $4$ [S1, unofficial].
- Window size: Spatial window for local variance calculation. Typical value: $3 \times 3$ [S1, unofficial].

**Steps**
1. **Calculate Local Variance**: Compute the variance $\sigma^2$ of the transmission color $\mathbf{C}_{\text{trans}}$ within a $3 \times 3$ window centered at the current pixel [S1, unofficial].
2. **Compute Gating Factor**: Calculate the gating factor $g$ using the variance $\sigma^2$ and hyperparameter $k$. The specific formula for $g$ is not detailed in the provided sources, but it is referenced as Eq. (5) in the original work [S1, unofficial]. The factor $g$ is designed to be low in high-variance (specular) regions and high in low-variance (transmission) regions.
3. **Apply Partial Stop-Gradient**: Compute the gated transmission color $\tilde{\mathbf{C}}_{\text{trans}}$ using the formula:
   $$
   \tilde{\mathbf{C}}_{\text{trans}} = (1 - g)\cdot \texttt{sg}(\mathbf{C}_{\text{trans}}) + g\cdot \mathbf{C}_{\text{trans}}
   $$
   where $\texttt{sg}(\cdot)$ is the stop-gradient operator [S1, unofficial].
4. **Forward Pass**: Use $\tilde{\mathbf{C}}_{\text{trans}}$ for final image composition. Note that $\tilde{\mathbf{C}}_{\text{trans}} \equiv \mathbf{C}_{\text{trans}}$ during the forward pass to avoid visual artifacts [S1, unofficial].
5. **Backward Pass**: The gradient of the loss $L$ with respect to $\mathbf{C}_{\text{trans}}$ is scaled by $g$. Specifically, $\frac{\partial L}{\partial \mathbf{C}_{\text{trans}}}$ is multiplied by $g$ before being propagated to earlier layers [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def specular_aware_gradient_gating(C_trans, k=4, window_size=3):
    """
    Applies Specular-Aware Gradient Gating to the transmission color.
    
    Args:
        C_trans: Tensor of transmission colors (H, W, 3).
        k: Gating strength hyperparameter.
        window_size: Size of the local window for variance calculation.
        
    Returns:
        C_trans_gated: Gated transmission color for forward pass.
        grad_C_trans: Modified gradient for backward pass.
    """
    # Step 1: Calculate local variance
    # Compute variance of C_trans in a window_size x window_size neighborhood
    # This can be done using a convolution with a uniform kernel or a sliding window
    variance = compute_local_variance(C_trans, window_size)
    
    # Step 2: Compute gating factor g
    # The exact formula for g is not provided in the sources, 
    # but it depends on variance and k. 
    # Based on the description, g should be low for high variance (specular) and high for low variance.
    # A common form for such gating is g = 1 / (1 + k * variance) or similar.
    # Since the exact formula is missing, we assume a placeholder function.
    g = compute_gating_factor(variance, k)
    
    # Step 3: Apply partial stop-gradient
    # Forward pass: C_trans_gated is identical to C_trans
    C_trans_gated = C_trans
    
    # Backward pass: Scale gradients by g
    # In a differentiable framework, this is implemented by stopping the gradient 
    # for the (1-g) part and allowing it for the g part.
    # C_trans_gated = (1 - g) * stop_gradient(C_trans) + g * C_trans
    # The gradient of C_trans_gated w.r.t C_trans is g.
    
    return C_trans_gated, g
```

**Edge Cases and Pitfalls**
- **Forward Pass Identity**: It is critical that $\tilde{\mathbf{C}}_{\text{trans}} \equiv \mathbf{C}_{\text{trans}}$ in the forward pass to avoid visual artifacts [S1, unofficial].
- **Hyperparameter Sensitivity**: The performance peaks at $k=4$, offering an optimal tradeoff between suppressing specular artifacts and preserving transmission details [S1, unofficial]. Deviating from this value may lead to suboptimal results.
- **Variance Calculation**: The local variance must be computed correctly over the specified window size to accurately identify specular regions.

## Operating characteristics
- **Time Complexity**: Adds overhead for local variance calculation and gating factor computation. The variance calculation is $O(N \cdot W^2)$ where $N$ is the number of pixels and $W$ is the window size.
- **Memory Complexity**: Requires storing the gating factor $g$ for each pixel or Gaussian contribution.
- **Performance**: At $k=4$, the method offers the best tradeoff between suppressing specular artifacts and preserving transmission details, as evaluated over transparent regions across eight scenes [S1, unofficial].
- **Strengths**: Effectively reduces specular artifacts in transparent regions without compromising transmission details.
- **Limitations**: Requires careful tuning of the hyperparameter $k$. The exact formula for the gating factor is not provided, which may hinder implementation.

## Background
Specular-Aware Gradient Gating was introduced in the context of joint reflection-transmission modeling with Gaussian Splatting [S1, unofficial]. It addresses the challenge of simultaneously reconstructing surface reflections and the transmission behind the surface, a task where baseline methods often fail [S1, unofficial]. The method is part of the RT-Splatting framework [S1, unofficial].

## Sources
[S1] RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting, https://github.com/WoojunePark/splat_papers/issues/71, fetched 2026-10-27 (unofficial)
