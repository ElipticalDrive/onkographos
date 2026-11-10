# Differentiable Quantization for Neural Fields

Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable quantization reduces the memory footprint of neural radiance fields by compressing grid values to single bytes during the optimization process, thereby preventing the quality degradation associated with post-training quantization [S1]. Unlike standard scalar or uniform quantization which are applied after training, this method integrates quantization into the training loop using a straight-through estimator, ensuring the model adapts to the low-precision representation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Raw grid values $\tilde{\mathbf{t}}$ (densities, diffuse colors, or features) from the neural field.
*   **Output:** Quantized grid values $\tilde{\mathbf{t}}'$ suitable for storage and rendering.

**Data Structures:**
*   Grid values are stored as single bytes (8-bit integers) during optimization and rendering.
*   The quantization function maps continuous values to discrete indices in $[0, 2^8 - 1]$.

**Parameters:**
*   $m$: Scaling factor for affine mapping.
    *   $m = 14$ for densities (computed using exponential nonlinearity) [S1].
    *   $m = 7$ for diffuse colors and features [S1].
*   Bit-depth: 8 bits ($2^8 = 256$ levels) [S1].

**Steps:**
1.  **Sigmoid Mapping:** Map the raw grid values $\tilde{\mathbf{t}}$ to the range $[0, 1]$ using a sigmoid function $\sigma$:
    $$ \tilde{\mathbf{t}}_{norm} = \sigma(\tilde{\mathbf{t}}) $$
2.  **Quantization:** Apply the quantization function $q(x)$ to $\tilde{\mathbf{t}}_{norm}$. This function rounds the value to the nearest discrete level and uses a straight-through estimator to allow gradients to pass through the non-differentiable rounding operation:
    $$ q(x) = x + \mathcal{V}\left(\frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x\right) $$
    where $\mathcal{V}$ is the straight-through estimator operator, which returns the gradient of the identity function for the forward pass but allows gradients to flow through the rounding term in the backward pass [S1].
3.  **Affine Mapping:** Map the quantized value back to the range $[-m, m]$ using the scaling factor $m$:
    $$ \tilde{\mathbf{t}}' = 2m \cdot q(\tilde{\mathbf{t}}_{norm}) - m $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def differentiable_quantize(x, m):
    """
    Quantizes grid values to single bytes during optimization.
    
    Args:
        x (torch.Tensor): Raw grid values.
        m (float): Scaling factor for affine mapping (14 for density, 7 for color/features).
        
    Returns:
        torch.Tensor: Quantized grid values in range [-m, m].
    """
    # Step 1: Sigmoid mapping to [0, 1]
    x_norm = torch.sigmoid(x)
    
    # Step 2: Quantization with straight-through estimator
    # Calculate the quantized value (rounded to nearest byte level)
    # (2^8 - 1) = 255
    quantized_idx = torch.round(x_norm * 255)
    quantized_val = quantized_idx / 255.0
    
    # Straight-through estimator: forward pass uses quantized_val,
    # backward pass uses gradient of x_norm (identity)
    # In PyTorch, this is achieved by detaching the quantized value for the forward pass
    # and using the original x_norm for the backward pass gradient calculation.
    quantized_val_st = x_norm.detach() + (quantized_val - x_norm).detach() + x_norm - x_norm.detach()
    # Simplified: just use x_norm for gradient flow, but return quantized_val for forward
    # The formula in the paper is: q(x) = x + V(rounded - x)
    # V acts as identity for gradients.
    # So forward: quantized_val. Backward: gradient flows through x_norm.
    
    # Step 3: Affine mapping to [-m, m]
    # t' = 2 * m * q(x) - m
    t_prime = 2.0 * m * quantized_val - m
    
    return t_prime
```

**Edge Cases and Pitfalls:**
*   **Post-training Quantization:** Simply quantizing grid values after training causes a mismatch between the optimized model and the rendering model, leading to significant quality drops [S1].
*   **Gradient Flow:** The non-differentiable rounding operation must be handled via a straight-through estimator to allow gradients to flow during backpropagation [S1].

## Subtopics

## Operating characteristics
*   **Memory:** Reduces grid memory consumption by storing values as single bytes (8-bit) instead of higher precision floats [S1].
*   **Performance:** Enables real-time view synthesis in unbounded scenes by reducing memory bandwidth requirements [S1].
*   **Quality:** Maintains rendering quality comparable to full-precision models when quantization is integrated into training, avoiding the quality drop seen in post-training quantization [S1].

## Background
*   **Origin:** Introduced in the MERF (Memory-Efficient Radiance Fields) paper for real-time view synthesis in unbounded scenes [S1].
*   **Development:** The method addresses the memory bottleneck of neural radiance fields by combining sigmoid mapping, byte quantization, and affine scaling within the optimization loop [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
