# Differentiable Quantization for Neural Fields

Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable quantization reduces the memory footprint of neural radiance fields by mapping continuous grid values to single-byte discrete levels during the training process [S1]. Unlike post-training quantization, which applies rounding after convergence and causes significant quality degradation due to the mismatch between the optimized model and the rendered one, this method integrates quantization into the optimization loop [S1]. It sets itself apart from sibling methods like [Scalar Quantization](5726-scalar-quantization.md) by ensuring the model adapts to the quantization noise via gradient flow, rather than suffering from static discretization errors.

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous grid parameters $\tilde{\mathbf{t}}$ (densities, colors, features) optimized during training.
*   **Output:** Quantized grid values $\tilde{\mathbf{t}}'$ stored in memory, which are used for rendering.

**Data Structures:**
*   Grid values are stored as single bytes (8-bit integers).
*   The quantization function maps inputs to the range $[0, 2^8 - 1]$ before conversion.

**Parameters:**
*   $m$: Scaling factor for affine mapping.
    *   $m = 14$ for densities (which use an exponential nonlinearity) [S1].
    *   $m = 7$ for diffuse colors and features [S1].
*   Bit depth: 8 bits (single byte) [S1].

**Steps:**
1.  **Sigmoid Mapping:** Map the continuous grid value $\tilde{\mathbf{t}}$ to the range $[0, 1]$ using a sigmoid function $\sigma(\cdot)$ [S1].
2.  **Quantization:** Apply the byte quantization function $q(x)$ to the sigmoid output. This involves rounding to the nearest integer level and subtracting the original value to create a residual [S1].
3.  **Affine Mapping:** Map the quantized index back to the target range $[-m, m]$ using the formula:
    $$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$
    [S1]
4.  **Interpolation and Summation:** Use $\tilde{\mathbf{t}}'$ for linear interpolation and summation as per the standard neural field architecture. Note that non-linearities (e.g., exponential for density) are applied *after* this step [S1].

**Formula:**
The quantization function $q(x)$ is defined as:
$$ q(x) = x + \mathcal{V}\left(\frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x\right) $$
where $\mathcal{V}$ denotes the straight-through estimator (STE) [S1]. The term inside $\mathcal{V}$ represents the difference between the rounded value and the original value. The STE allows gradients to pass through the non-differentiable floor operation by treating the derivative of the rounding operation as 1 for the input $x$ and 0 for the residual term [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def differentiable_quantize(t, m):
    """
    Quantizes grid values to single bytes during optimization.
    
    Args:
        t (torch.Tensor): Continuous grid values.
        m (float): Scaling factor for the output range [-m, m].
        
    Returns:
        torch.Tensor: Quantized values in range [-m, m].
    """
    # Step 1: Sigmoid mapping to [0, 1]
    x = torch.sigmoid(t)
    
    # Step 2: Quantization with Straight-Through Estimator
    # Calculate the rounded index scaled to [0, 1]
    # floor((2^8 - 1) * x + 0.5) / (2^8 - 1)
    levels = 2**8 - 1
    rounded = torch.floor(levels * x + 0.5) / levels
    
    # Straight-Through Estimator:
    # Gradient flows through 'x' directly, ignoring the rounding operation.
    # q(x) = x + (rounded - x) where gradient of (rounded - x) is 0 w.r.t x
    q_x = x + (rounded - x).detach()
    
    # Step 3: Affine mapping to [-m, m]
    # q_x is in [0, 1], so 2*m*q_x - m maps it to [-m, m]
    t_quantized = 2 * m * q_x - m
    
    return t_quantized

# Example usage for densities (m=14) and colors (m=7)
# density_quantized = differentiable_quantize(density_grid, m=14)
# color_quantized = differentiable_quantize(color_grid, m=7)
```

**Edge Cases and Pitfalls:**
*   **Post-Training Quantization:** Simply quantizing grid values after training causes a mismatch between the optimized model and the rendering model, leading to a drop in rendering quality [S1].
*   **Gradient Flow:** The straight-through estimator is critical; without it, gradients cannot flow through the non-differentiable rounding operation, preventing the model from adapting to the quantization noise [S1].
*   **Non-linearity Order:** Non-linearities (such as the exponential function for density) must be applied *after* linear interpolation and summation of the quantized values, not before [S1].

## Subtopics

### [Quantization-Aware Fine-Tuning](11387-quantization-aware-fine-tuning.md)
A post-clustering optimization step to regain quality lost during vector quantization by re-optimizing scene parameters under reduced bit-rate constraints.

## Operating characteristics
*   **Memory Reduction:** Reduces grid memory consumption by storing values as single bytes (8-bit) instead of higher-precision floats [S1].
*   **Quality:** Maintains rendering quality comparable to full-precision models when quantization is integrated into training, avoiding the quality drops seen in post-training quantization [S1].
*   **Performance:** No significant runtime overhead is introduced during rendering, as the quantized values are used directly for interpolation [S1].
*   **Limitations:** The method is specific to grid-based neural fields and requires careful tuning of the scaling factor $m$ for different attribute types (density vs. color) [S1].

## Background
The method was introduced in the MERF (Memory-Efficient Radiance Fields) paper, which addresses the challenge of real-time view synthesis in unbounded scenes with limited memory [S1]. The authors observed that standard quantization techniques applied after training led to significant quality degradation, motivating the development of a differentiable quantization approach that allows the model to learn robust representations under quantization constraints [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
