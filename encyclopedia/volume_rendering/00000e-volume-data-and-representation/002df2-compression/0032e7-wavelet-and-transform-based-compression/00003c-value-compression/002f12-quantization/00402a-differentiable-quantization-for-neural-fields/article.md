# Differentiable Quantization for Neural Fields

Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable quantization reduces the memory footprint of neural field representations (such as 3D Gaussian Splatting or NeRF grids) by compressing stored values to single bytes during the optimization process [S1]. Unlike post-training quantization, which applies discretization after convergence and causes significant quality degradation due to the mismatch between the optimized continuous model and the rendered discrete model, this method integrates quantization into the training loop [S1]. It sets itself apart from standard scalar or uniform quantization by using a differentiable approximation (straight-through estimator) that allows gradients to flow through the non-differentiable rounding operation, enabling the network to adapt its weights to the low-precision representation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Continuous grid values $\tilde{\mathbf{t}}$ (densities, colors, features) during optimization.
*   **Output:** Quantized grid values $\tilde{\mathbf{t}}'$ stored in memory, and gradients $\frac{\partial L}{\partial \tilde{\mathbf{t}}}$ passed back to the network.

**Data Structures**
*   Grid values are stored as single bytes (8-bit integers) in the range $[0, 255]$ internally, but mapped to a specific physical range $[-m, m]$ for rendering.
*   The quantization function $q(x)$ maps a float $x \in [0, 1]$ to a byte.

**Parameters**
*   $m$: Scaling factor for the affine mapping.
    *   $m = 14$ for densities (which use an exponential nonlinearity).
    *   $m = 7$ for diffuse colors and features [S1].
*   Bit-depth: 8 bits ($2^8 - 1 = 255$) [S1].

**Steps**
1.  **Sigmoid Mapping:** Map the continuous grid value $\tilde{\mathbf{t}}$ to the range $[0, 1]$ using a sigmoid function $\sigma$:
    $$ x = \sigma(\tilde{\mathbf{t}}) $$
2.  **Quantization:** Apply the quantization function $q(x)$ to map $x$ to a byte. This involves rounding to the nearest integer level and normalizing back to $[0, 1]$.
3.  **Affine Mapping:** Map the quantized value to the target range $[-m, m]$:
    $$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$
4.  **Gradient Flow:** Use the straight-through estimator to allow gradients to pass through the non-differentiable rounding operation in $q(x)$.

**Formula**
The quantization function $q(x)$ is defined as:
$$ q(x) = x + \mathcal{V}\left(\frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x\right) $$
where $\mathcal{V}$ is the straight-through estimator operator, which returns the gradient of the identity function for the forward pass but allows the gradient of the rounding term to pass in the backward pass (effectively treating the operation as $x$ for gradient computation) [S1].

The final stored value is:
$$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def sigmoid(x):
    return torch.sigmoid(x)

def quantize_byte(x):
    """
    Quantizes a value x in [0, 1] to a byte and returns the normalized float.
    Uses straight-through estimator for gradients.
    """
    # Round to nearest integer level [0, 255]
    # (2^8 - 1) = 255
    levels = 255
    rounded = torch.floor((levels * x) + 0.5)
    normalized = rounded / levels
    
    # Straight-through estimator:
    # Forward pass: return normalized
    # Backward pass: gradient flows through x (identity)
    # In PyTorch, this is often implemented via detach or custom autograd function.
    # Here we simulate the straight-through behavior:
    # The term (normalized - x) has gradient, but we stop gradient for the forward value.
    # However, the source formula implies the output is the normalized value,
    # but gradients flow through the original x.
    
    # Standard STE implementation:
    # forward: normalized
    # backward: gradient of x
    return normalized + (x - x.detach())

def differentiable_quantization(t, m):
    """
    Applies differentiable quantization to grid values t.
    
    Args:
        t: Continuous grid values (floats)
        m: Scaling factor (14 for density, 7 for color/features)
        
    Returns:
        t_prime: Quantized values in range [-m, m]
    """
    # Step 1: Sigmoid mapping to [0, 1]
    x = sigmoid(t)
    
    # Step 2 & 3: Quantize and affine map
    # q(x) returns value in [0, 1]
    q_x = quantize_byte(x)
    
    # Affine map to [-m, m]
    t_prime = 2 * m * q_x - m
    
    return t_prime
```

**Edge Cases and Pitfalls**
*   **Post-Training Quantization:** Simply quantizing grid values after training causes a mismatch between the optimized model and the rendering model, leading to a significant drop in rendering quality [S1].
*   **Nonlinearity Application:** The non-linearities (e.g., exponential for density) are applied *after* linear interpolation and summation of the quantized values, not before [S1].
*   **Occupancy:** Content is only baked in regions considered occupied by the proposal MLP during training to avoid wasting memory on empty space [S1].

## Subtopics

### [Quantization-Aware Fine-Tuning](11387-quantization-aware-fine-tuning.md)
A post-clustering optimization step to recover quality lost during vector quantization by simulating quantization errors during training.

## Operating characteristics
*   **Memory Reduction:** Reduces grid value storage to single bytes (8-bit) per dimension [S1].
*   **Performance:** Enables real-time view synthesis in unbounded scenes by reducing memory bandwidth and storage requirements [S1].
*   **Quality:** Maintains high rendering quality comparable to full-precision models when integrated during optimization, avoiding the quality drops seen in post-training quantization [S1].
*   **Limitations:** Requires careful tuning of the scaling factor $m$ for different attribute types (density vs. color) [S1].

## Background
This method was introduced in the MERF (Memory-Efficient Radiance Fields) paper for real-time view synthesis in unbounded scenes [S1]. It addresses the high memory consumption of neural field representations by enabling efficient storage and rendering of large-scale scenes.

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
