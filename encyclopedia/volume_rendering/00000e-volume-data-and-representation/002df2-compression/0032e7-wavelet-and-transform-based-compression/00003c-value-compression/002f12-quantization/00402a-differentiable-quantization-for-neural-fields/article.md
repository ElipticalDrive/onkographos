# Differentiable Quantization for Neural Fields
Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable quantization reduces the memory footprint of neural field representations (such as 3D Gaussian Splatting or NeRF grids) by mapping continuous parameters to single-byte discrete values during the optimization process [S1]. Unlike post-training quantization, which causes significant quality degradation due to the mismatch between the optimized continuous model and the discrete inference model, this method integrates quantization into the training loop using a straight-through estimator [S1]. It sets itself apart from sibling methods like [Vector Quantization](907-vector-quantization.md) by using uniform scalar quantization with specific affine scaling rather than learned codebooks, and from [Scalar Quantization](5726-scalar-quantization.md) by enforcing differentiability during training to preserve gradient flow [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous grid values $\tilde{\mathbf{t}}$ (densities, colors, features) optimized during training.
*   **Output:** Quantized grid values $\tilde{\mathbf{t}}'$ stored as single bytes (uint8) for render-time memory efficiency [S1].

**Data Structures:**
*   The grid stores values as 8-bit unsigned integers.
*   The mapping involves three stages: sigmoid normalization, integer rounding, and affine reconstruction [S1].

**Parameters:**
*   $m$: Scaling factor for the affine reconstruction.
    *   $m = 14$ for densities (which use an exponential nonlinearity) [S1].
    *   $m = 7$ for diffuse colors and features [S1].
*   Bit-depth: 8 bits ($2^8 - 1 = 255$) [S1].

**Steps:**
1.  **Normalization:** Apply a sigmoid function $\sigma(\cdot)$ to the continuous grid values $\tilde{\mathbf{t}}$ to map them to the range $[0, 1]$.
    $$ x = \sigma(\tilde{\mathbf{t}}) $$
2.  **Quantization:** Apply the differentiable quantization function $q(x)$ to map the normalized value to a single byte index. This function uses a straight-through estimator (STE) to allow gradients to pass through the non-differentiable rounding operation [S1].
    $$ q(x) = x + \text{stop-gradient}\left( \frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x \right) $$
    Here, $\lfloor \cdot \rfloor$ denotes the floor function, and $\text{stop-gradient}(\cdot)$ prevents gradients from flowing through the rounding term during backpropagation [S1].
3.  **Affine Reconstruction:** Map the quantized value back to the target range $[-m, m]$ using the scaling factor $m$.
    $$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$
4.  **Storage:** Store $\tilde{\mathbf{t}}'$ as an 8-bit integer in the grid [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def differentiable_quantize(t: torch.Tensor, m: float) -> torch.Tensor:
    """
    Quantizes continuous grid values to single bytes during optimization.
    
    Args:
        t: Continuous grid values (densities, colors, or features).
        m: Scaling factor (14 for densities, 7 for colors/features).
        
    Returns:
        Quantized values mapped to [-m, m] range, compatible with gradient flow.
    """
    # Step 1: Normalize to [0, 1] using sigmoid
    x = torch.sigmoid(t)
    
    # Step 2: Quantize to 8-bit integer range [0, 255]
    # Calculate the rounded integer index
    # (2^8 - 1) = 255
    quantized_index = torch.floor((255 * x) + 0.5)
    
    # Normalize back to [0, 1]
    quantized_normalized = quantized_index / 255.0
    
    # Straight-Through Estimator:
    # The forward pass uses the quantized value.
    # The backward pass uses the original continuous value x.
    # We compute the difference and stop gradients on the rounding error.
    quantized_value = x + torch.stop_gradient(quantized_normalized - x)
    
    # Step 3: Affine mapping to [-m, m]
    # Formula: 2 * m * quantized_value - m
    result = 2.0 * m * quantized_value - m
    
    return result
```

**Edge Cases and Pitfalls:**
*   **Post-Training Quantization:** Simply quantizing grid values after training completes leads to a mismatch between the optimized model and the rendering model, causing a significant drop in rendering quality [S1].
*   **Gradient Flow:** The rounding operation is non-differentiable. Without the straight-through estimator, gradients cannot flow through the quantization step, preventing the network from learning appropriate values that survive quantization [S1].
*   **Nonlinearity Application:** The non-linearities defined in the radiance field equations (e.g., exponential for density) are applied *after* linear interpolation and summation of the quantized values, not before [S1].

## Subtopics

### [Quantization-Aware Fine-Tuning](11387-quantization-aware-fine-tuning.md)
A post-clustering optimization step to recover quality lost during vector quantization by fine-tuning while simulating low-precision quantization.

## Operating characteristics
*   **Memory Reduction:** Reduces memory consumption by mapping each grid dimension to a single byte (8 bits) [S1].
*   **Performance:** Enables real-time view synthesis in unbounded scenes by reducing render-time memory bandwidth and storage requirements [S1].
*   **Quality:** Maintains high rendering quality comparable to full-precision models when integrated during training, avoiding the quality drops associated with post-training quantization [S1].
*   **Limitations:** The method is specific to grid-based neural field representations and requires careful selection of the scaling factor $m$ for different attribute types (density vs. color) [S1].

## Background
This method was introduced in the MERF (Memory-Efficient Radiance Fields) paper for real-time view synthesis in unbounded scenes [S1]. It addresses the high memory requirements of neural radiance fields by enabling efficient storage and rendering of large-scale scenes [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
