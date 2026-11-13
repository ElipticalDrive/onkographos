# Specular-Aware Gradient Gating

Also called: Partial stop-gradient mechanism for transmission

## Overview
Specular-Aware Gradient Gating suppresses specular artifacts in semi-transparent surfaces while preserving transmission details during 3D Gaussian Splatting training [S1, unofficial]. It differs from sibling methods by modulating gradients specifically for the transmission color component using a local variance-based gating factor, rather than applying global stop-gradient operations or deferred back-propagation [S1, unofficial].

## How it works
**Inputs:**
- Original transmission color $\mathbf{C}_{\text{trans}}$ (per pixel or per Gaussian contribution).
- Local neighborhood of transmission colors for variance calculation.

**Outputs:**
- Gated transmission color $\tilde{\mathbf{C}}_{\text{trans}}$ used for final composition.
- Gradients scaled by gating factor $g$ during back-propagation.

**Data Structures:**
- No specific complex data structures required beyond standard image buffers. A 3x3 window is used for local variance calculation.

**Parameters:**
- Gating strength hyperparameter $k$: 4 [S1, unofficial].
- Local variance window size: 3 × 3 [S1, unofficial].

**Steps:**
1. **Calculate Local Variance:** Compute the variance of transmission colors within a 3 × 3 window centered at the current pixel. Let this variance be $\text{Var}$.
2. **Compute Gating Factor:** Calculate the gating factor $g$ based on the variance $\text{Var}$ and the hyperparameter $k$. The exact formula for $g$ from Eq. (5) is not detailed in the provided sources, but it is a function of $\text{Var}$ and $k$ [S1, unofficial].
3. **Apply Partial Stop-Gradient:** Compute the gated transmission color $\tilde{\mathbf{C}}_{\text{trans}}$ using the formula:
   $$ \tilde{\mathbf{C}}_{\text{trans}} = (1 - g)\cdot \texttt{sg}(\mathbf{C}_{\text{trans}}) + g\cdot \mathbf{C}_{\text{trans}} $$
   where $\texttt{sg}(\cdot)$ is the stop-gradient operator [S1, unofficial].
4. **Forward Pass:** Use $\tilde{\mathbf{C}}_{\text{trans}}$ for final composition. Note that $\tilde{\mathbf{C}}_{\text{trans}} \equiv \mathbf{C}_{\text{trans}}$ during the forward pass to avoid visual artifacts [S1, unofficial].
5. **Backward Pass:** Gradients are scaled by $g$ during back-propagation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def specular_aware_gradient_gating(C_trans, k=4, window_size=3):
    """
    Applies specular-aware gradient gating to transmission color.
    
    Args:
        C_trans: Original transmission color (tensor).
        k: Gating strength hyperparameter.
        window_size: Size of the local variance window.
        
    Returns:
        C_trans_gated: Gated transmission color for forward pass.
        grad_scale: Scaling factor for backward pass gradients.
    """
    # Step 1: Calculate local variance
    # Note: The exact formula for variance calculation is not provided in the sources.
    # Assuming a standard variance calculation over a window.
    # This is a placeholder for the actual implementation of Eq. 5's variance part.
    # In practice, this would involve a convolution or pooling operation.
    variance = calculate_local_variance(C_trans, window_size)
    
    # Step 2: Compute gating factor g
    # Note: The exact formula for g from Eq. 5 is not provided in the sources.
    # It is a function of variance and k.
    # Placeholder: g = f(variance, k)
    # Based on the context, g likely increases with variance to suppress specular artifacts.
    # A common form might be g = 1 / (1 + exp(-k * (variance - threshold))) or similar.
    # However, without the exact formula, we cannot implement this precisely.
    # We will assume a placeholder function for g.
    g = compute_gating_factor(variance, k)
    
    # Step 3: Apply partial stop-gradient
    # Forward pass: C_trans_gated is equivalent to C_trans
    # Backward pass: gradients are scaled by g
    # The formula given is: C_trans_gated = (1 - g) * sg(C_trans) + g * C_trans
    # Since sg(C_trans) stops gradients, the gradient of C_trans_gated w.r.t C_trans is g.
    
    # For forward pass, we just return C_trans
    C_trans_gated = C_trans
    
    # For backward pass, the gradient scaling factor is g
    grad_scale = g
    
    return C_trans_gated, grad_scale

def calculate_local_variance(C_trans, window_size):
    """
    Calculates local variance of transmission colors.
    This is a placeholder implementation.
    """
    # Not given in the sources.
    # Typically, this would involve computing the mean and variance over a sliding window.
    pass

def compute_gating_factor(variance, k):
    """
    Computes the gating factor g based on variance and k.
    This is a placeholder implementation.
    """
    # Not given in the sources.
    # The exact formula from Eq. 5 is not provided.
    pass
```

**Edge Cases and Pitfalls:**
- Forward pass must remain identical to original transmission ($\tilde{\mathbf{C}}_{\text{trans}} \equiv \mathbf{C}_{\text{trans}}$) to avoid visual artifacts [S1, unofficial].
- The exact formula for the gating factor $g$ is not provided in the sources, which may lead to implementation difficulties [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Performance peaks at $k = 4$, offering an optimal tradeoff between suppressing specular artifacts and preserving transmission details [S1, unofficial].
- **Strengths:** Effectively suppresses specular artifacts while preserving transmission details [S1, unofficial].
- **Limitations:** The exact formula for the gating factor is not provided, which may hinder implementation [S1, unofficial].

## Background
Specular-Aware Gradient Gating is part of the RT-Splatting method, which jointly models reflection and transmission with Gaussian Splatting [S1, unofficial]. The method was introduced to address the challenge of simultaneously reconstructing surface reflections and the transmission behind the surface, which baseline methods often fail to do [S1, unofficial].

## Sources
[S1] RT-Splatting: Joint Reflection-Transmission Modeling with Gaussian Splatting, https://github.com/WoojunePark/splat_papers/issues/71, fetched 2026-10-27 (unofficial)
