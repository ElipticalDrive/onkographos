# Specular-Aware Gradient Gating

Also called: Partial stop-gradient mechanism for transmission

## Overview
Specular-Aware Gradient Gating suppresses specular artifacts in semi-transparent volume rendering while preserving transmission details during 3D Gaussian Splatting training [S1, unofficial]. It differs from standard back-propagation by modulating the gradient flow of the transmission color component using a local variance-based gating factor, ensuring the forward pass remains visually identical to the original model [S1, unofficial].

## How it works
**Inputs:**
- $\mathbf{C}_{\text{trans}}$: The transmission color component for a pixel, computed during the forward pass.
- $k$: Gating strength hyperparameter.
- $W$: Window size for local variance calculation.

**Outputs:**
- $\tilde{\mathbf{C}}_{\text{trans}}$: The gated transmission color used for final composition (forward pass).
- Gradients $\frac{\partial L}{\partial \mathbf{C}_{\text{trans}}}$: Modulated gradients for back-propagation.

**Data Structures:**
- No specific persistent data structures are required beyond the standard per-pixel buffers used in the tile-and-bin pipeline. The gating factor $g$ is computed per-pixel or per-tile based on local neighborhood statistics.

**Steps:**
1. **Calculate Local Variance:** Compute the variance of the transmission color $\mathbf{C}_{\text{trans}}$ within a local window of size $W \times W$ centered at the pixel. Let this variance be $\sigma^2$. [S1, unofficial]
2. **Compute Gating Factor:** Calculate the gating factor $g$ using the variance $\sigma^2$ and the hyperparameter $k$. The exact functional form mapping $\sigma^2$ and $k$ to $g$ is not explicitly detailed in the provided text, but it is defined in Equation (5) of the source [S1, unofficial].
3. **Apply Partial Stop-Gradient:** Compute the gated transmission color $\tilde{\mathbf{C}}_{\text{trans}}$ for the forward pass using the formula below. This ensures $\tilde{\mathbf{C}}_{\text{trans}} \equiv \mathbf{C}_{\text{trans}}$ in the forward direction, but scales the backward gradients by $g$ [S1, unofficial].

**Formula:**
$$
\tilde{\mathbf{C}}_{\text{trans}} = (1 - g)\cdot \texttt{sg}(\mathbf{C}_{\text{trans}}) + g\cdot \mathbf{C}_{\text{trans}}
$$
where:
- $\tilde{\mathbf{C}}_{\text{trans}}$ is the gated transmission color for final composition.
- $g$ is the gating factor ($0 \le g \le 1$).
- $\texttt{sg}(\cdot)$ is the stop-gradient operator.
- $\mathbf{C}_{\text{trans}}$ is the original transmission color.

**Parameters:**
- Gating strength hyperparameter $k$: 4 [S1, unofficial].
- Local variance window size $W$: $3 \times 3$ [S1, unofficial].

**Pitfalls:**
- **Forward Pass Identity:** The forward pass must remain identical to the original transmission ($\tilde{\mathbf{C}}_{\text{trans}} \equiv \mathbf{C}_{\text{trans}}$) to avoid visual artifacts. The gating mechanism only affects the backward pass gradients [S1, unofficial].
- **Hyperparameter Sensitivity:** Performance peaks at $k=4$. Deviations may lead to either insufficient suppression of specular artifacts or loss of transmission details [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def specular_aware_gradient_gating(C_trans, k=4, window_size=3):
    """
    Applies Specular-Aware Gradient Gating to the transmission color.
    
    Args:
        C_trans: Tensor of transmission colors (H, W, 3).
        k: Gating strength hyperparameter.
        window_size: Size of the local variance window (e.g., 3).
        
    Returns:
        C_trans_gated: The gated transmission color for forward pass.
        grad_scale: The scaling factor for backward gradients.
    """
    # Step 1: Calculate local variance
    # Compute variance of C_trans within a window_size x window_size neighborhood
    # This requires a sliding window or convolution operation
    variance = compute_local_variance(C_trans, window_size)
    
    # Step 2: Compute gating factor g
    # The exact formula for g from variance and k is not provided in the text.
    # It is assumed to be a function g(variance, k) that maps to [0, 1].
    # Based on the context, higher variance (specular highlights) likely leads to lower g.
    g = compute_gating_factor(variance, k)
    
    # Step 3: Apply partial stop-gradient
    # Forward pass: C_trans_gated is identical to C_trans
    # Backward pass: gradients are scaled by g
    # In automatic differentiation frameworks, this is implemented by:
    # 1. Creating a new tensor for the forward pass that is a copy of C_trans
    # 2. Detaching C_trans from the computation graph for the stop-gradient term
    # 3. Combining them such that the gradient flow is modulated by g
    
    # Implementation note:
    # C_trans_gated = (1 - g) * C_trans.detach() + g * C_trans
    # However, since C_trans.detach() has no gradient, the gradient of C_trans_gated
    # w.r.t C_trans is only through the second term: g * C_trans.
    # Thus, the effective gradient scaling is g.
    
    # To ensure forward pass identity:
    C_trans_gated = C_trans.clone() # Forward pass is identical
    
    # For backward pass, we return the scaling factor g
    # The framework will handle the gradient scaling if implemented via
    # a custom autograd function or by using the formula in the loss computation.
    
    return C_trans_gated, g

def compute_local_variance(tensor, window_size):
    """
    Computes local variance for each pixel in a tensor using a sliding window.
    """
    # Pad the tensor to handle boundaries
    pad = window_size // 2
    padded_tensor = torch.nn.functional.pad(tensor, (pad, pad, pad, pad, 0, 0), mode='reflect')
    
    # Use unfold to get all windows
    # Assuming tensor is (H, W, 3), unfold works on (N, C, H, W)
    # Reshape for unfold
    N, H, W, C = tensor.shape
    unfolded = padded_tensor.unfold(0, window_size, 1).unfold(1, window_size, 1)
    # unfolded shape: (H, W, window_size, window_size, 3)
    
    # Compute mean and variance along the window dimensions
    mean = unfolded.mean(dim=(2, 3))
    variance = ((unfolded - mean.unsqueeze(2).unsqueeze(3)) ** 2).mean(dim=(2, 3))
    
    return variance

def compute_gating_factor(variance, k):
    """
    Computes the gating factor g from variance and k.
    The exact formula is not provided in the sources.
    This is a placeholder implementation based on typical gating mechanisms.
    """
    # Placeholder: The actual formula from Eq. (5) is missing.
    # A common form might be g = 1 / (1 + variance * k) or similar.
    # Since the exact formula is not given, we cannot implement this correctly.
    # [G] Placeholder implementation: g = 1 / (1 + variance * k)
    # This is a guess and should be replaced with the actual formula from Eq. (5).
    return 1.0 / (1.0 + variance * k)
```

## Operating characteristics
- **Time Complexity:** Adds overhead for local variance calculation ($O(W^2)$ per pixel) and gating factor computation. The variance calculation can be optimized using separable filters or integral images.
- **Memory Complexity:** Requires storage for the variance map and gating factor map, proportional to the image resolution.
- **Performance:** Peaks at $k=4$, offering an optimal trade-off between suppressing specular artifacts and preserving transmission details [S1, unofficial].
- **Strengths:** Effectively reduces specular artifacts in semi-transparent scenes without degrading transmission quality.
- **Limitations:** Requires careful tuning of $k$; suboptimal values may lead to visual artifacts or loss of detail.

## Background
Specular-Aware Gradient Gating was introduced in the context of RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting [S1, unofficial]. It addresses the challenge of simultaneously reconstructing surface reflections and transmission in semi-transparent scenes, where standard gradient propagation can lead to artifacts.

## Sources
[S1] RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting, https://github.com/WoojunePark/splat_papers/issues/71, fetched 2026-10-27 (unofficial)
