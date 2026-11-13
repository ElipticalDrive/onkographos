# Differentiable Quantization for Neural Fields
Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable quantization reduces the memory footprint of neural field representations (such as 3D Gaussian Splatting or NeRF grids) by mapping continuous parameters to single-byte integers during the training process [S1]. Unlike post-training quantization, which causes quality degradation due to the mismatch between the optimized continuous model and the discrete inference model, this method integrates quantization into the optimization loop [S1]. It sets itself apart from standard scalar quantization by using a straight-through estimator to allow gradients to flow through the non-differentiable rounding operation, ensuring the network learns values that are robust to discretization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous grid values $\tilde{\mathbf{t}}$ (densities, colors, or features) produced by the neural field.
*   **Output:** Quantized values $\tilde{\mathbf{t}}'$ stored in the grid, which are single-byte integers mapped to a specific physical range.

**Data Structures:**
*   The grid stores values as 8-bit unsigned integers (bytes).
*   Each byte represents one of $2^8 - 1 = 255$ discrete levels.

**Parameters:**
*   $m$: The scaling factor for the affine mapping.
    *   $m = 14$ for densities (which use an exponential nonlinearity) [S1].
    *   $m = 7$ for diffuse colors and features [S1].
*   $N = 2^8 - 1 = 255$: The number of quantization levels [S1].

**Steps:**
1.  **Sigmoid Mapping:** Map the continuous input $\tilde{\mathbf{t}}$ to the range $[0, 1]$ using a sigmoid function $\sigma(\cdot)$ [S1].
2.  **Quantization:** Apply the quantization function $q(x)$ to the sigmoided value. This function rounds the value to the nearest discrete level and uses a straight-through estimator for gradients [S1].
3.  **Affine Mapping:** Map the quantized byte value back to the target range $[-m, m]$ [S1].

**Formula:**
The final quantized value $\tilde{\mathbf{t}}'$ is computed as:
$$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$

The quantization function $q(x)$ is defined as:
$$ q(x) = x + \mathcal{V}\left(\frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x\right) $$
where $\mathcal{V}$ denotes the straight-through estimator (STE). In the forward pass, the term inside $\mathcal{V}$ is zeroed out, effectively returning the rounded value $\frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1}$. In the backward pass, the gradient flows through the identity term $x$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def sigmoid(x):
    return torch.sigmoid(x)

def quantize(x):
    """
    Quantizes a value in [0, 1] to a single byte using straight-through estimator.
    Forward: Rounds to nearest 255th.
    Backward: Gradient passes through the original float value.
    """
    # Round to nearest integer level in [0, 255]
    # (2^8 - 1) is 255
    levels = 255
    rounded = torch.round(x * levels) / levels
    # Clamp to [0, 1] to handle boundary precision issues
    rounded = torch.clamp(rounded, 0.0, 1.0)
    
    # Straight-through estimator:
    # Forward pass uses 'rounded', backward pass uses 'x'
    # torch.stop_gradient is not standard in PyTorch, using detach() for forward
    # and direct x for backward via custom logic or assuming autograd handles 
    # the identity if we structure it as:
    # q(x) = x + (rounded - x).detach()
    # This ensures grad(x) = 1 + 0 = 1
    
    q_x = x + (rounded - x).detach()
    return q_x

def differentiable_quantization(t, m):
    """
    Applies differentiable quantization to grid values.
    
    Args:
        t: Continuous grid values (float)
        m: Scaling factor (14 for density, 7 for color/features)
        
    Returns:
        t_prime: Quantized values in range [-m, m]
    """
    # 1. Sigmoid to [0, 1]
    t_sigmoid = sigmoid(t)
    
    # 2. Quantize to byte
    t_quantized = quantize(t_sigmoid)
    
    # 3. Affine map to [-m, m]
    # Formula: 2*m * q(x) - m
    t_prime = 2 * m * t_quantized - m
    
    return t_prime
```

**Edge Cases and Pitfalls:**
*   **Post-training Quantization:** Simply quantizing grid values after training causes a significant drop in rendering quality because the optimized model does not match the quantized inference model [S1].
*   **Precision:** The method relies on the straight-through estimator to bypass the non-differentiability of the rounding operation [S1].

## Subtopics

### [Quantization-Aware Fine-Tuning](11387-quantization-aware-fine-tuning.md)
A post-clustering optimization step that recovers quality lost during vector quantization by fine-tuning the scene representation while simulating low-precision quantization during the forward pass.

## Operating characteristics
*   **Memory:** Reduces memory consumption by storing each grid dimension as a single byte [S1].
*   **Performance:** Enables real-time view synthesis in unbounded scenes by reducing memory bandwidth requirements [S1].
*   **Quality:** Maintains high rendering quality comparable to full-precision models when integrated during optimization, avoiding the quality drops associated with post-training quantization [S1].
*   **Limitations:** Requires quantization to be performed during the optimization phase; it cannot be easily applied as a post-processing step without retraining or fine-tuning [S1].

## Background
This method was introduced in the MERF (Memory-Efficient Radiance Fields) paper, which focuses on real-time view synthesis in unbounded scenes [S1]. The authors identified that standard quantization after training led to mismatches between the training and rendering models, necessitating the integration of quantization into the optimization loop [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
