# Differentiable Quantization for Neural Fields

Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
This method reduces the render-time memory footprint of neural radiance fields by quantizing grid-based feature values to single bytes during the optimization process [S1]. Unlike post-training quantization, which causes quality degradation due to mismatch between the optimized model and the inference model, this approach integrates quantization into the training loop using a straight-through estimator [S1]. It sets itself apart from sibling methods like [Vector Quantization](907-vector-quantization.md) by using uniform byte-level quantization on grid values rather than clustering parameters into codebooks, and from [Scalar Quantization](5726-scalar-quantization.md) by explicitly handling the non-differentiable nature of rounding via gradient approximation during training [S1].

## How it works
**Inputs and Outputs:**
- Input: Continuous grid values $\tilde{\mathbf{t}}$ (densities, diffuse colors, or features) optimized during training.
- Output: Quantized grid values $\tilde{\mathbf{t}}'$ stored as single bytes, mapped to a specific range $[-m, m]$.

**Data Structures:**
- The grid stores values as single bytes (8-bit integers).
- Each dimension of the grid (density, color, features) is quantized independently.

**Parameters:**
- $m$: Scaling factor for the affine mapping.
  - $m = 14$ for densities (computed using exponential nonlinearity) [S1].
  - $m = 7$ for diffuse colors and features [S1].
- Bit depth: 8 bits (single byte) [S1].

**Steps:**
1. **Sigmoid Mapping:** Map the continuous grid value $\tilde{\mathbf{t}}$ to the range $[0, 1]$ using a sigmoid function $\sigma$:
   $$ x = \sigma(\tilde{\mathbf{t}}) $$
2. **Quantization:** Apply the byte quantization function $q(x)$ to map $x$ to the nearest discrete level in $[0, 1]$. The function uses a straight-through estimator to allow gradients to flow through the non-differentiable rounding operation [S1]:
   $$ q(x) = x + \mathcal{V}\left(\frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x\right) $$
   where $\mathcal{V}$ denotes the straight-through estimator (the gradient of the identity function is passed through the stop-gradient operation on the rounded term) [S1].
3. **Affine Mapping:** Map the quantized value back to the target range $[-m, m]$:
   $$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$
4. **Storage:** Store $\tilde{\mathbf{t}}'$ as a single byte in the grid [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def sigmoid_quantize(t, m):
    """
    Quantizes grid values to single bytes during optimization.
    
    Args:
        t: Continuous grid values (tensor).
        m: Scaling factor (14 for densities, 7 for colors/features).
        
    Returns:
        Quantized values mapped to [-m, m].
    """
    # Step 1: Sigmoid mapping to [0, 1]
    x = torch.sigmoid(t)
    
    # Step 2: Quantization with straight-through estimator
    # Calculate the rounded value
    rounded = torch.floor((2**8 - 1) * x + 0.5) / (2**8 - 1)
    
    # Straight-through estimator: gradient flows through 'x', not 'rounded'
    # q(x) = x + stop_gradient(rounded - x)
    q_x = x + (rounded - x).detach()
    
    # Step 3: Affine mapping to [-m, m]
    t_prime = 2 * m * q_x - m
    
    return t_prime

# Example usage for density (m=14) and color (m=7)
# density_quantized = sigmoid_quantize(density_grid, m=14)
# color_quantized = sigmoid_quantize(color_grid, m=7)
```

**Edge Cases and Pitfalls:**
- Simply quantizing grid values after training causes a mismatch between the optimized model and the rendering model, leading to a drop in rendering quality [S1].
- Quantization must be performed during optimization to allow the model to adapt to the quantization error [S1].
- The non-linearities in the radiance field computation (e.g., exponential for density) are applied after linearly interpolating and summing the quantized values, not before [S1].

## Subtopics

## Operating characteristics
- **Memory:** Reduces memory consumption by storing each grid dimension as a single byte [S1].
- **Performance:** Enables real-time view synthesis in unbounded scenes by reducing memory bandwidth requirements [S1].
- **Quality:** Maintains rendering quality comparable to full-precision models when quantization is integrated into training [S1].
- **Limitations:** Requires careful selection of scaling factors ($m$) for different data types (density vs. color) [S1].

## Background
This method was introduced in the MERF (Memory-Efficient Radiance Fields) paper for real-time view synthesis in unbounded scenes [S1]. It addresses the high memory consumption of neural radiance fields by quantizing grid values to single bytes during optimization [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
