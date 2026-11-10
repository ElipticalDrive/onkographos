# Differentiable Quantization for Neural Fields

Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable quantization reduces the memory footprint of neural field representations (such as 3D Gaussian Splatting or NeRF grids) by compressing stored values to single bytes during the optimization process. Unlike post-training quantization, which applies discretization after convergence and causes significant quality degradation due to the mismatch between the optimized continuous model and the rendered discrete model, this method integrates quantization into the training loop. It sets itself apart from standard [Scalar Quantization](5726-scalar-quantization.md) by using a straight-through estimator to allow gradients to flow through the non-differentiable rounding operation, ensuring the network adapts to the low-precision representation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous grid values $\tilde{\mathbf{t}}$ (densities, colors, or features) optimized during training.
*   **Output:** Quantized grid values $\tilde{\mathbf{t}}'$ stored as single bytes (uint8) but treated as floating-point values for gradient computation.

**Data Structures:**
*   The grid stores values as 8-bit integers (uint8) at render time.
*   During training, values are mapped to the range $[0, 1]$ before quantization.

**Parameters:**
*   $m$: Scaling factor for affine mapping.
    *   $m = 14$ for densities (due to exponential nonlinearity).
    *   $m = 7$ for diffuse colors and features [S1].
*   $B = 2^8 - 1 = 255$: Number of quantization levels [S1].

**Steps:**
1.  **Sigmoid Mapping:** Map the continuous value $\tilde{\mathbf{t}}$ to $[0, 1]$ using a sigmoid function $\sigma(\cdot)$:
    $$ x = \sigma(\tilde{\mathbf{t}}) $$
2.  **Quantization:** Apply the quantization function $q(x)$ to round $x$ to the nearest of 256 levels. This function uses a straight-through estimator (STE) to pass gradients through the `stop-gradient` operation:
    $$ q(x) = x + \text{stop-gradient}\left( \frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x \right) $$
    Here, $\lfloor \cdot \rfloor$ denotes the floor function. The term inside `stop-gradient` represents the difference between the quantized value and the original continuous value. The gradient of $q(x)$ with respect to $x$ is effectively 1, allowing backpropagation to proceed as if the operation were identity [S1].
3.  **Affine Mapping:** Map the quantized value $q(x)$ from $[0, 1]$ to $[-m, m]$:
    $$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$
4.  **Storage:** Store $\tilde{\mathbf{t}}'$ as a single byte in the grid.
5.  **Rendering:** During rendering, linearly interpolate the stored byte values and apply subsequent non-linearities (e.g., exponential for density) as defined by the specific neural field formulation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def differentiable_quantize(t, m):
    """
    Quantizes continuous tensor t to single byte precision during training.
    
    Args:
        t: Continuous tensor of shape (..., D)
        m: Scaling factor (14 for density, 7 for color/features)
        
    Returns:
        Quantized tensor t_prime with gradients flowing through the quantization.
    """
    # Step 1: Sigmoid mapping to [0, 1]
    x = torch.sigmoid(t)
    
    # Step 2: Quantization with Straight-Through Estimator
    # Calculate quantized value: round to nearest of 256 levels in [0, 1]
    # (2^8 - 1) is 255
    quantized_x = torch.round(x * 255) / 255.0
    
    # Straight-Through Estimator: 
    # Forward pass uses quantized_x, backward pass uses x (identity gradient)
    # torch.stop_gradient is simulated by detaching the quantized value 
    # and adding the residual to the original input.
    t_prime = x + (quantized_x - x).detach()
    
    # Step 3: Affine mapping to [-m, m]
    t_prime = 2 * m * t_prime - m
    
    return t_prime

def render_step(grid_values):
    """
    Simulates the rendering step where interpolated values are processed.
    Note: In practice, grid_values are uint8. This function assumes 
    they have been de-quantized to float for the non-linearities.
    """
    # Linear interpolation and summation happen here (implementation depends on specific field)
    # Then apply non-linearities. For density, this is often exponential.
    pass
```

**Edge Cases and Pitfalls:**
*   **Post-Training Quantization:** Simply quantizing grid values after training converges causes a mismatch between the optimized model and the rendered model, leading to significant quality drops [S1].
*   **Precision:** The use of `stop-gradient` is critical. Without it, the gradient of the rounding operation is zero almost everywhere, preventing learning [S1].
*   **Range Selection:** The choice of $m$ is crucial. Incorrect scaling can lead to saturation or underutilization of the 8-bit range [S1].

## Subtopics

### [Quantization-Aware Fine-Tuning](11387-quantization-aware-fine-tuning.md)
A post-clustering optimization step to recover quality lost during vector quantization. It differs by optimizing scene parameters at reduced bit-rates using simulated quantization errors during training, whereas differentiable quantization applies scalar byte quantization directly to grid values during the main optimization phase.

## Operating characteristics
*   **Memory:** Reduces memory consumption by storing each grid dimension as a single byte (8 bits) instead of 32-bit floats (4 bytes), achieving a 4x compression ratio per value [S1].
*   **Performance:** No significant runtime overhead compared to standard floating-point rendering, as the quantization is baked into the grid storage. The rendering pipeline simply reads uint8 values and interpolates them [S1].
*   **Quality:** Maintains high rendering quality when integrated into training, avoiding the artifacts associated with post-training quantization [S1].
*   **Limitations:** The method is specific to grid-based or splat-based neural fields where values can be sigmoid-mapped to [0,1]. It does not directly apply to unbounded or sparse data without appropriate bounding or proposal networks [S1].

## Background
This method was introduced in the MERF (Memory-Efficient Radiance Fields) paper, which focuses on real-time view synthesis in unbounded scenes. The key insight is that render-time memory is a bottleneck for large-scale scenes, and quantizing grid values to bytes is effective if done differentially during optimization to avoid quality loss [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
