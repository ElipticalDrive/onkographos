# Exponential Decay Blur Filter
Also called: Exponential Smoothing Blur, IIR Blur
## Overview
A linear blur filter that applies exponential decay smoothing along the outermost two dimensions of an input tensor [S1, unofficial]. It serves as a post-processing step in separable ping-pong blur pipelines to reduce high-frequency noise and aliasing artifacts in rendered images [S1, unofficial]. Unlike Gaussian blurs, it uses an infinite impulse response (IIR) structure, allowing for efficient computation with constant memory per pass regardless of kernel width [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A tensor $X$ of shape $(..., H, W)$, typically representing a rendered image or depth map [S1, unofficial].
- Output: A smoothed tensor $Y$ of the same shape as $X$ [S1, unofficial].

**Parameters:**
- Smoothing parameter $\alpha$: Controls the decay rate. Smaller $\alpha$ results in more smoothing; larger $\alpha$ results in less dependence on previous values and thus less smoothing [S2, unofficial].
- Typical value: $\alpha = 0.8$ is used for most methods [S2, unofficial]. For IDR (Iterative Deepening Representation), $\alpha = 0.3$ is used due to higher variance between plot points [S2, unofficial].

**Data Structures:**
- The algorithm operates in-place or uses a single temporary buffer $Y$ initialized as a copy of $X$ [S1, unofficial].

**Steps:**
The filter is separable, processing the last two dimensions independently. Each dimension undergoes a forward pass followed by a backward pass.

1. Initialize $Y = X$ [S1, unofficial].
2. Forward pass along the last dimension (width $W$):
   For $i$ from $0$ to $W-2$:
   $$Y[..., i+1] \leftarrow Y[..., i+1] + \alpha \cdot Y[..., i]$$
3. Backward pass along the last dimension:
   For $i$ from $W-2$ down to $0$:
   $$Y[..., i] \leftarrow Y[..., i] + \alpha \cdot Y[..., i+1]$$
4. Forward pass along the second-to-last dimension (height $H$):
   For $i$ from $0$ to $H-2$:
   $$Y[..., i+1, :] \leftarrow Y[..., i+1, :] + \alpha \cdot Y[..., i, :]$$
5. Backward pass along the second-to-last dimension:
   For $i$ from $H-2$ down to $0$:
   $$Y[..., i, :] \leftarrow Y[..., i, :] + \alpha \cdot Y[..., i+1, :]$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exponential_decay_blur(X, alpha):
    """
    Applies exponential decay smoothing to the last two dimensions of tensor X.
    
    Args:
        X: Input tensor of shape (..., H, W).
        alpha: Smoothing parameter (0 < alpha <= 1).
        
    Returns:
        Y: Smoothed tensor of the same shape as X.
    """
    # Copy input to output buffer
    Y = X.clone()
    
    # Dimensions
    H = Y.shape[-2]
    W = Y.shape[-1]
    
    # Forward pass along last dimension (width)
    for i in range(W - 1):
        Y[..., i + 1] = Y[..., i + 1] + alpha * Y[..., i]
        
    # Backward pass along last dimension (width)
    for i in range(W - 2, -1, -1):
        Y[..., i] = Y[..., i] + alpha * Y[..., i + 1]
        
    # Forward pass along second-to-last dimension (height)
    for i in range(H - 1):
        Y[..., i + 1, :] = Y[..., i + 1, :] + alpha * Y[..., i, :]
        
    # Backward pass along second-to-last dimension (height)
    for i in range(H - 2, -1, -1):
        Y[..., i, :] = Y[..., i, :] + alpha * Y[..., i + 1, :]
        
    return Y
```

**Edge Cases and Pitfalls:**
- Boundary conditions: The loops run up to `shape-1`, so boundary pixels are updated by their neighbors but do not propagate values outside the tensor bounds [S1, unofficial].
- Precision: Accumulation of floating-point errors is possible in very long sequences, though typically negligible for image dimensions [G].
- Degenerate input: If $H=1$ or $W=1$, the corresponding loop range is empty, and the pass is skipped, which is correct behavior [G].

## Subtopics

### Exponential Moving Average Smoothing
A variant where the smoothing is defined recursively as $\hat{v}_i = \alpha v_i + (1 - \alpha) \hat{v}_{i-1}$ for $i > 1$, with $\hat{v}_1 = v_1$ [S2, unofficial]. This formulation explicitly maintains a running average rather than the additive decay used in the main filter. It is used in training convergence analysis, such as in IDR, where $\alpha = 0.3$ is preferred due to higher variance between plot points [S2, unofficial]. For other methods, $\alpha = 0.8$ is used [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot (H + W))$ for a tensor of size $N \times H \times W$, where $N$ is the batch size or number of channels. The separable nature reduces complexity from $O(N \cdot H \cdot W \cdot K^2)$ for a 2D convolutional kernel of size $K$.
- **Memory Complexity:** $O(1)$ auxiliary memory per pass if implemented in-place, or $O(N \cdot H \cdot W)$ if a copy is required.
- **Performance:** Efficient on GPUs due to simple arithmetic operations and coalesced memory access patterns [S1, unofficial].
- **Strengths:** Low computational cost, easy to implement, effective for smoothing noisy data like depth priors or rendered images [S1, unofficial].
- **Limitations:** Introduces directional bias if not applied symmetrically (though the ping-pong approach mitigates this); not a true Gaussian blur, so frequency response differs [G].
- **When to use:** Post-processing for noise reduction in neural rendering pipelines, particularly when computational efficiency is critical [S1, unofficial].
- **When not to use:** When precise frequency cutoff characteristics are required, or when isotropic smoothing is strictly needed without the IIR artifacts [G].

## Background
The method originates from implementations in neural rendering pipelines, specifically in the context of NeRFingMVS, where it is used to smooth depth priors and confidences [S1, unofficial]. The concept of exponential smoothing is well-established in time-series analysis and signal processing, but its application as a spatial filter in volume rendering post-processing is a specific adaptation for handling noise in learned representations [S1, unofficial]. The variant using exponential moving averages is cited in the context of training neural lumigraph representations using meta learning, where it is used to smooth convergence metrics [S2, unofficial].

## Sources
[S1] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S2] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
