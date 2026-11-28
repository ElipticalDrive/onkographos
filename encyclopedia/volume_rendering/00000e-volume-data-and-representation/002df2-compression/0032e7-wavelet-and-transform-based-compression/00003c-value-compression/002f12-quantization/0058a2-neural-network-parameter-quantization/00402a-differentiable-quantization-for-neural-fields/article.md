# Differentiable Quantization for Neural Fields

Also called: MERF Quantization, Quantization for Memory Reduction

## Overview
Differentiable quantization reduces the render-time memory footprint of neural radiance fields by compressing grid values into single bytes during the optimization phase [S1]. Unlike post-training quantization, which causes quality degradation due to the mismatch between the optimized continuous model and the discrete inference model, this method integrates quantization directly into the training loop [S1]. It sets itself apart from sibling methods like Global Quantization by applying per-component affine scaling and sigmoid mapping to handle the specific dynamic ranges of densities versus colors [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous grid values $\tilde{\mathbf{t}}$ representing densities, diffuse colors, or features, optimized during training [S1].
*   **Output:** Quantized grid values $\tilde{\mathbf{t}}'$ stored as single bytes, affinely mapped to the target range for rendering [S1].

**Data Structures:**
*   Grid values are stored as 8-bit unsigned integers (bytes) [S1].
*   The quantization function $q(x)$ maps a float in $[0, 1]$ to one of $2^8 - 1 = 255$ discrete levels [S1].

**Numbered Steps:**
1.  **Sigmoid Mapping:** Apply a sigmoid function $\sigma(\cdot)$ to the continuous grid values $\tilde{\mathbf{t}}$ to map them into the range $[0, 1]$ [S1].
2.  **Quantization:** Apply the differentiable quantization function $q(x)$ to the sigmoided values [S1].
3.  **Affine Mapping:** Map the quantized result to the range $[-m, m]$ using the specified scaling factor [S1].
4.  **Interpolation and Summation:** During rendering, linearly interpolate the quantized grid values and apply subsequent non-linearities (e.g., exponential for density) as defined in the base neural field architecture [S1].

**Formulas:**
The combined quantization and affine mapping operation is defined as:
$$ \tilde{\mathbf{t}}' = 2m \cdot q(\sigma(\tilde{\mathbf{t}})) - m $$
where $\sigma(\cdot)$ is the sigmoid function, $m$ is the scaling factor, and $q(x)$ is the byte quantization function [S1]. The quantization function implements a straight-through estimator (STE) to allow gradients to flow through the non-differentiable rounding operation:
$$ q(x) = x + \text{stop\_gradient}\left(\frac{\lfloor (2^8 - 1)x + 1/2 \rfloor}{2^8 - 1} - x\right) $$
where $\text{stop\_gradient}(\cdot)$ prevents gradients from flowing through the rounding term during backpropagation [S1].

**Parameters:**
*   $m$: The scaling factor for the affine mapping.
    *   $m = 14$ for densities (which are computed using an exponential nonlinearity) [S1].
    *   $m = 7$ for diffuse colors and features [S1].
*   Bit depth: Fixed at 8 bits (1 byte) per value [S1].

**Edge Cases and Pitfalls:**
*   **Post-Training Quantization:** Simply quantizing grid values after training creates a mismatch between the optimized model and the rendering model, leading to significant quality drops [S1]. Quantization must be performed during optimization [S1].
*   **Gradient Flow:** The non-differentiable rounding operation requires the straight-through estimator to allow gradients to propagate during backpropagation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def differentiable_quantize(t, m):
    """
    Quantizes continuous grid values to single bytes during optimization.
    
    Args:
        t (torch.Tensor): Continuous grid values.
        m (float): Scaling factor (14 for density, 7 for color/features).
        
    Returns:
        torch.Tensor: Quantized and affinely mapped values.
    """
    # Step 1: Sigmoid mapping to [0, 1]
    t_sigmoid = torch.sigmoid(t)
    
    # Step 2: Quantization to 256 levels (0 to 255)
    levels = 2**8 - 1
    quantized_index = torch.floor(levels * t_sigmoid + 0.5)
    quantized_normalized = quantized_index / levels
    
    # Step 3: Straight-Through Estimator (STE)
    # Forward pass uses quantized_normalized
    # Backward pass uses t_sigmoid (gradient flows through the identity connection)
    quantized_ste = t_sigmoid + (quantized_normalized - t_sigmoid).detach()
    
    # Step 4: Affine mapping to [-m, m]
    t_quantized = 2 * m * quantized_ste - m
    
    return t_quantized
```

## Subtopics

### [Quantization-Aware Fine-Tuning](11387-quantization-aware-fine-tuning.md)
A post-clustering optimization step to recover quality lost during vector quantization by fine-tuning with simulated low-precision quantization.

### [Learned Bitwidth Quantization (LBQ)](16775-learned-bitwidth-quantization-lbq.md)
A framework that treats bitwidth as a learnable variable, dynamically allocating bits per component based on content sensitivity, unlike the fixed byte size of this method.

## Operating characteristics
*   **Memory Reduction:** Reduces render-time memory consumption by storing grid values as single bytes instead of full-precision floats [S1].
*   **Quality:** Maintains high rendering quality when integrated into the optimization process, avoiding the degradation seen in post-training quantization [S1].
*   **Limitations:** The method is specific to grid-based neural fields and requires careful selection of the scaling parameter $m$ for different data types [S1].

## Background
This method was introduced in the MERF (Memory-Efficient Radiance Fields) paper for real-time view synthesis in unbounded scenes [S1]. It addresses the high memory consumption of neural radiance fields by quantizing grid values during optimization, allowing for efficient storage and rendering [S1].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
