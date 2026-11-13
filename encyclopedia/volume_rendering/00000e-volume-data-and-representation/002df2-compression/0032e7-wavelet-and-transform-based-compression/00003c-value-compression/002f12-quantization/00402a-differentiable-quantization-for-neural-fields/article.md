# Differentiable Quantization for Neural Fields
Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable quantization reduces the memory footprint of neural field representations (such as 3D Gaussian Splatting or NeRF grids) by mapping continuous parameters to single-byte integers during the optimization phase [S1]. Unlike post-training quantization, which causes significant rendering quality degradation due to the mismatch between the optimized continuous model and the discrete inference model, this method integrates quantization into the training loop [S1]. It sets itself apart from sibling methods like [Vector Quantization](907-vector-quantization.md) by operating on scalar grid values directly using a straight-through estimator, rather than clustering parameters into codebooks [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Continuous grid values $\tilde{\mathbf{t}}$ (densities, colors, or features) optimized during training.
*   **Output:** Quantized values $\tilde{\mathbf{t}}'$ stored in the grid, which are single-byte integers mapped back to a specific real-valued range for rendering.

**Data Structures**
*   The grid stores values as 8-bit unsigned integers (bytes).
*   Each dimension of the neural field (e.g., density, RGB) is quantized independently to a single byte [S1].

**Parameters**
*   $m$: The scaling factor for the affine mapping range $[-m, m]$.
    *   $m = 14$ for densities (which use an exponential nonlinearity) [S1].
    *   $m = 7$ for diffuse colors and features [S1].
*   $B = 2^8 - 1 = 255$: The number of quantization levels [S1].

**Steps**
1.  **Sigmoid Mapping:** Map the continuous value $x$ (from $\tilde{\mathbf{t}}$) to the range $[0, 1]$ using a sigmoid function $\sigma(x)$ [S1].
2.  **Quantization:** Apply the differentiable quantization function $q(\cdot)$ to the sigmoided value. This function rounds the value to the nearest integer level and uses a straight-through estimator to pass gradients [S1].
3.  **Affine Mapping:** Map the quantized byte value back to the target range $[-m, m]$ [S1].
4.  **Storage:** Store the resulting value in the grid. Note that non-linearities (like exponential activation for density) are applied *after* linear interpolation and summation of these quantized values during rendering [S1].

**Formula**
The quantization function $q(x)$ is defined as:
$$ q(x) = x + \mathcal{V}\left(\frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x\right) $$
where $\mathcal{V}(\cdot)$ is the straight-through estimator (STE) operator, which returns the gradient of the identity function for the input $x$ while the forward pass computes the rounded discrete value [S1].

The final stored value $\tilde{\mathbf{t}}'$ for a dimension is:
$$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$
where $\sigma(\cdot)$ is the sigmoid function [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def sigmoid(x):
    return torch.sigmoid(x)

def straight_through_estimator(x):
    """
    Forward pass: returns x.
    Backward pass: passes gradient through as if it were identity.
    """
    return x + (torch.round(x) - x).detach()

def differentiable_quantize(x, m):
    """
    Quantizes continuous value x to a single byte and maps to [-m, m].
    
    Args:
        x: Continuous input value (tensor).
        m: Scaling factor for the output range.
        
    Returns:
        Quantized and affine-mapped value.
    """
    # Step 1: Sigmoid mapping to [0, 1]
    x_sigmoid = sigmoid(x)
    
    # Step 2: Quantization to byte (0-255)
    # Calculate the discrete level: floor((255 * x) + 0.5)
    # Divide by 255 to get back to [0, 1] range
    # Apply STE to allow gradients to flow through the rounding
    levels = 255
    quantized_level = torch.floor(levels * x_sigmoid + 0.5)
    quantized_normalized = quantized_level / levels
    
    # Use STE: forward pass uses the rounded value, backward pass uses x_sigmoid
    quantized_st = quantized_normalized + (x_sigmoid - quantized_normalized).detach()
    
    # Step 3: Affine mapping to [-m, m]
    # Range is [0, 1] -> [-m, m]
    # Formula: 2 * m * value - m
    result = 2 * m * quantized_st - m
    
    return result

# Example usage for density (m=14) and color (m=7)
density_quantized = differentiable_quantize(density_input, m=14)
color_quantized = differentiable_quantize(color_input, m=7)
```

**Edge Cases and Pitfalls**
*   **Post-training Quantization:** Simply quantizing grid values after training creates a mismatch between the optimized model and the rendering model, leading to a significant drop in rendering quality [S1].
*   **Non-linearity Placement:** Non-linearities (e.g., exponential for density) must be applied after interpolation and summation of the quantized values, not before [S1].
*   **Occupied Regions:** Content should only be baked in regions considered occupied by the proposal MLP during training to avoid artifacts [S1].

## Subtopics

### [Quantization-Aware Fine-Tuning](11387-quantization-aware-fine-tuning.md)
A post-clustering optimization step to recover quality lost during vector quantization by fine-tuning while simulating low-precision quantization.

## Operating characteristics
*   **Memory Reduction:** Reduces memory consumption by quantizing each grid dimension to a single byte [S1].
*   **Quality:** Maintains rendering quality comparable to full-precision models when integrated during optimization, avoiding the quality drops seen in post-training quantization [S1].
*   **Performance:** No significant runtime performance overhead is mentioned, as the quantization is a storage optimization [S1].
*   **Limitations:** Requires integration into the training pipeline; cannot be easily applied to already-trained models without retraining or fine-tuning [S1].

## Background
This method was introduced in the MERF (Memory-Efficient Radiance Fields) paper for real-time view synthesis in unbounded scenes [S1]. It addresses the high memory consumption of neural field representations by enabling single-byte storage for grid values while preserving rendering quality through differentiable optimization [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
