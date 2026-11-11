# Differentiable Quantization for Neural Fields

Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable Quantization for Neural Fields is a technique that reduces the memory footprint of neural radiance fields by compressing grid values into single bytes during the training process itself. Unlike post-training quantization, which applies discretization after convergence and often leads to significant rendering quality degradation due to distribution mismatch, this method integrates quantization into the optimization loop. It sets itself apart from standard scalar quantization by using a straight-through estimator to allow gradients to flow through the non-differentiable rounding operation, ensuring the network adapts to the quantization noise.

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous grid values $\tilde{\mathbf{t}}$ (densities, colors, features) produced by the neural field or proposal MLP.
*   **Output:** Quantized grid values $\tilde{\mathbf{t}}'$ stored in memory, and the gradient signal for backpropagation.

**Data Structures:**
*   Grid values are stored as single bytes (8-bit integers).
*   The quantization maps values to the range $[0, 255]$ before affine scaling.

**Parameters:**
*   $m$: The scaling factor for the affine mapping.
    *   $m = 14$ for densities (which use an exponential nonlinearity).
    *   $m = 7$ for diffuse colors and features [S1].
*   $B = 2^8 - 1 = 255$: The number of quantization levels minus one.

**Steps:**
1.  **Nonlinear Mapping:** Apply a sigmoid function $\sigma(\cdot)$ to map the raw grid values $\tilde{\mathbf{t}}$ to the interval $[0, 1]$. This step is crucial for bounding the values before quantization [S1].
2.  **Quantization:** Apply the quantization function $q(x)$ to the sigmoid-mapped values. This function rounds the input to the nearest discrete level and uses a straight-through estimator (STE) for gradients.
3.  **Affine Scaling:** Map the quantized value back to the target range $[-m, m]$ using the formula:
    $$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$
    This allows the network to represent negative values and scales the magnitude appropriately for the specific attribute type [S1].
4.  **Interpolation and Summation:** The quantized values $\tilde{\mathbf{t}}'$ are then linearly interpolated and summed according to the field's architecture (e.g., Equation 6 in the source) before subsequent nonlinearities are applied [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def differentiable_quantize(x, m):
    """
    Quantizes grid values to single bytes during optimization.
    
    Args:
        x: Continuous input tensor (raw grid values).
        m: Scaling factor for the output range [-m, m].
           Use 14 for densities, 7 for colors/features.
           
    Returns:
        quantized_x: Quantized tensor in range [-m, m].
    """
    # Step 1: Map to [0, 1] using sigmoid
    x_sigmoid = torch.sigmoid(x)
    
    # Step 2: Quantize to byte using Straight-Through Estimator
    # The forward pass rounds to the nearest integer level
    # The backward pass passes gradients through the identity function
    
    # Calculate the quantized level index (0 to 255)
    # floor((255 * x) + 0.5) is equivalent to rounding to nearest integer
    levels = 255
    quantized_level = torch.floor(levels * x_sigmoid + 0.5)
    
    # Clip to valid range [0, 255] to handle boundary cases
    quantized_level = torch.clamp(quantized_level, 0, levels)
    
    # Normalize back to [0, 1]
    quantized_normalized = quantized_level / levels
    
    # Straight-Through Estimator:
    # Forward: use quantized_normalized
    # Backward: use x_sigmoid (identity gradient)
    # In PyTorch, this is achieved by detaching the quantized value 
    # for the forward pass but adding the original sigmoid value 
    # to the gradient flow, or using a custom autograd function.
    # Here we simulate the STE behavior:
    quantized_x_forward = quantized_normalized
    quantized_x_backward = x_sigmoid.detach() + (x_sigmoid - x_sigmoid.detach())
    
    # Note: In a real implementation, one would use a custom autograd Function
    # to ensure gradients flow correctly. The formula from the source is:
    # q(x) = x + stop_gradient(floor((2^8 - 1)x + 1/2)/(2^8 - 1) - x)
    # This simplifies to: q(x) = floor((2^8 - 1)x + 1/2)/(2^8 - 1) in forward,
    # but gradient of x is passed in backward.
    
    # Re-implementing strictly according to source formula for clarity:
    # q(x) = x + stop_gradient(quantized_normalized - x)
    # This is equivalent to returning quantized_normalized in forward,
    # and x in backward.
    
    # Let's use the standard STE pattern:
    # Forward: quantized_normalized
    # Backward: gradient of x_sigmoid is passed through
    
    # Step 3: Affine mapping to [-m, m]
    # t' = 2 * m * q(x) - m
    result = 2 * m * quantized_x_forward - m
    
    # To ensure gradients flow correctly for training, we must attach the 
    # gradient of the input x to the result. 
    # Since quantized_x_forward is a function of x_sigmoid, and x_sigmoid 
    # is a function of x, we need to preserve the gradient path.
    # The STE effectively replaces the gradient of the rounding operation 
    # with the identity.
    
    # Correct STE implementation in PyTorch:
    # q(x) = (x_sigmoid - x_sigmoid.detach()) + quantized_normalized
    # This adds the gradient of x_sigmoid to the quantized value.
    
    q_x = (x_sigmoid - x_sigmoid.detach()) + quantized_normalized
    result = 2 * m * q_x - m
    
    return result
```

**Edge Cases and Pitfalls:**
*   **Post-Training Quantization:** Simply quantizing grid values after training causes a mismatch between the optimized model and the rendering model, leading to a drop in rendering quality [S1].
*   **Precision:** The method relies on the straight-through estimator to handle the non-differentiable rounding. Without it, gradients cannot flow through the quantization step, preventing the network from adapting to the quantization noise [S1].
*   **Boundary Values:** The sigmoid function maps values to $[0, 1]$, but extreme values may saturate. The affine mapping then scales these to $[-m, m]$. The choice of $m$ is critical for preserving the dynamic range of the data [S1].

## Subtopics

### [Quantization-Aware Fine-Tuning](11387-quantization-aware-fine-tuning.md)
A post-clustering optimization step designed to regain reconstruction quality lost during the vector quantization (clustering) of scene parameters. It solves the problem of information loss inherent in compressing continuous neural field or Gaussian splatting parameters into discrete codebooks by re-optimizing the scene representation under the constraint of reduced bit-rates. Unlike post-training quantization, which applies quantization only after convergence, this method simulates quantization during the optimization process to allow the model to adapt its parameters to the specific errors introduced by the target precision, thereby minimizing the final reconstruction error.

## Operating characteristics
*   **Memory Reduction:** Reduces grid memory consumption by storing values as single bytes instead of floats (typically 4 bytes), achieving a 4x reduction in memory per grid value [S1].
*   **Performance:** No significant runtime overhead during rendering, as the quantization is baked into the stored grid values. The training time may increase slightly due to the additional operations, but the gradient flow is maintained via STE.
*   **Quality:** Maintains high rendering quality comparable to full-precision training, as the network adapts to the quantization noise during optimization [S1].
*   **Limitations:** The method is specific to grid-based neural fields. It requires careful tuning of the scaling factor $m$ for different attribute types (density vs. color) [S1].

## Background
Differentiable Quantization for Neural Fields was introduced in the MERF (Memory-Efficient Radiance Fields) paper. It addresses the challenge of rendering unbounded scenes with high fidelity while keeping memory usage low. The key insight is that quantization can be made differentiable using the straight-through estimator, allowing it to be integrated into the training process. This approach avoids the quality degradation associated with post-training quantization.

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
