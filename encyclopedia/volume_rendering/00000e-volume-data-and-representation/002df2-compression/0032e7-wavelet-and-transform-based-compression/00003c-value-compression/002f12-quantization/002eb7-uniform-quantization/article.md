# Uniform Quantization


## Overview
Uniform quantization is a lossy compression method that maps a large range of continuous or high-precision discrete data values to a limited set of discrete levels using constant spacing between levels [S1, S2, unofficial]. It solves the problem of reducing memory bandwidth and storage requirements for volume data, typically achieving a 4:1 compression ratio by representing 32-bit floating-point data with 8-bit integers [S1, S2, unofficial]. It differs from its siblings, such as [Non-uniform Quantization](5858-non-uniform-quantization.md), by using fixed step sizes regardless of the data distribution, which makes it simple to implement but potentially ineffective for data with wide dynamic ranges where important features may be lost [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume dataset $V$ with raw values $x$, minimum value $x_{min}$, maximum value $x_{max}$, and a target bit depth $b$ (e.g., 8 bits).
- Output: A quantized dataset $V'$ where each value is an integer in the range $[0, 2^b - 1]$.

**Data Structures:**
- The quantized data is stored as a linear array or grid of unsigned integers (e.g., `uint8` for 8-bit quantization).
- Metadata required for decoding includes $x_{min}$, $x_{max}$, and $b$.

**Parameters:**
- $b$: Target bit depth. Typical value: 8 [S1, S2, unofficial].
- $M$: Number of quantization levels, calculated as $M = 2^b$.
- $\phi$: Step size (quantization interval), calculated as $\phi = (x_{max} - x_{min}) / M$ [S2, unofficial].

**Steps:**
1. Determine the global minimum $x_{min}$ and maximum $x_{max}$ of the input volume data.
2. Calculate the number of levels $M = 2^b$.
3. Calculate the step size $\phi = (x_{max} - x_{min}) / M$.
4. For each voxel value $x$ in the input volume:
   a. Clamp $x$ to the range $[x_{min}, x_{max}]$.
   b. Compute the quantized index $q = \lfloor (x - x_{min}) / \phi \rfloor$.
   c. Clamp $q$ to the range $[0, M - 1]$.
   d. Store $q$ as the quantized value.
5. Store $x_{min}$, $x_{max}$, and $b$ alongside the quantized data for reconstruction.

**Formulas:**
The quantization error for uniformly distributed source data is measured as the mean squared error (MSE) [S2, unofficial]:
$$
\sigma^2 = \int_{x_{min}}^{x_{max}} \frac{1}{M} (x - \phi_i)^2 dx
$$
where $f(x) = \frac{1}{x_{max} - x_{min}}$ is the probability density function for uniform distribution [S2, unofficial].

For visualization tasks, the error measure incorporates an importance function $\alpha(x)$, typically the opacity transfer function provided by the user [S1, S2, unofficial]:
$$
\sigma^2 = E \int (x - \phi(x))^2 f(x) \alpha(x) dx
$$
where $f(x)$ characterizes the general source data distribution and $\alpha(x)$ is the importance function [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_quantize(volume, bit_depth=8):
    """
    Quantize a volume dataset using uniform quantization.
    
    Args:
        volume: 3D array of float values.
        bit_depth: Number of bits for quantization (default 8).
        
    Returns:
        quantized_volume: 3D array of uint8 values.
        min_val: Minimum value of the original volume.
        max_val: Maximum value of the original volume.
    """
    min_val = volume.min()
    max_val = volume.max()
    
    # Handle edge case where min_val equals max_val
    if min_val == max_val:
        return np.zeros_like(volume, dtype=np.uint8), min_val, max_val
        
    num_levels = 2 ** bit_depth
    step_size = (max_val - min_val) / num_levels
    
    # Normalize to [0, 1] range relative to min_val
    normalized = (volume - min_val) / step_size
    
    # Clamp to [0, num_levels - 1]
    quantized_indices = np.clip(normalized, 0, num_levels - 1)
    
    # Cast to uint8
    quantized_volume = quantized_indices.astype(np.uint8)
    
    return quantized_volume, min_val, max_val

def uniform_dequantize(quantized_volume, min_val, max_val, bit_depth=8):
    """
    Dequantize a uniform quantized volume.
    
    Args:
        quantized_volume: 3D array of uint8 values.
        min_val: Minimum value used during quantization.
        max_val: Maximum value used during quantization.
        bit_depth: Number of bits used during quantization.
        
    Returns:
        reconstructed_volume: 3D array of float values.
    """
    num_levels = 2 ** bit_depth
    step_size = (max_val - min_val) / num_levels
    
    # Convert back to float and scale
    reconstructed_volume = (quantized_volume.astype(np.float32) * step_size) + min_val
    
    return reconstructed_volume
```

**Edge Cases and Pitfalls:**
- If $x_{min} = x_{max}$, the step size becomes zero, leading to division by zero. This case should be handled by returning a constant volume [G].
- Values outside the $[x_{min}, x_{max}]$ range are clamped, which may introduce bias if the range is not representative of the entire dataset [S1, S2, unofficial].

## Subtopics

### [Non-uniform Quantization](5858-non-uniform-quantization.md)
Allocates more quantization levels to specific ranges of data values to enhance features and minimize distortion error, particularly useful for data with wide dynamic ranges. Choose this when uniform spacing causes important features to become invisible.

### [Scalar Quantization](5726-scalar-quantization.md)
General term for quantizing scalar values, which may include uniform, non-uniform, or adaptive methods.

### [Attribute Quantization](1912-attribute-quantization.md)
Quantization applied to specific attributes (e.g., color, normal) rather than the primary scalar field.

### [Chroma Subsampling](6512-chroma-subsampling.md)
Reduces color resolution, often used in conjunction with scalar quantization for texture or color data.

### [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md)
Quantization methods designed for neural implicit representations, allowing gradient-based optimization.

### [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md)
Compression techniques for neural networks representing implicit surfaces or fields.

## Operating characteristics
- **Time Complexity:** $O(N)$ for quantization and dequantization, where $N$ is the number of voxels.
- **Memory Complexity:** $O(N \cdot b)$ bits, where $b$ is the target bit depth.
- **Measured Performance:** Achieves a 4:1 compression ratio by representing 32-bit data with 8 bits [S1, S2, unofficial].
- **Strengths:** Simple implementation, low computational overhead, predictable memory usage.
- **Limitations:** Ineffective for data with wide dynamic ranges (e.g., turbulence flow) where important features may be lost if not quantized with care [S1, S2, unofficial]. Non-uniform or adaptive quantization is preferred in such cases to allocate more levels to ranges of interest [S1, S2, unofficial].
- **When to Use:** When data has a relatively uniform distribution and simplicity is prioritized over feature preservation.
- **When Not to Use:** When data has a wide dynamic range or non-uniform distribution, as uniform quantization may cause important features to become invisible [S1, S2, unofficial].

## Background
Uniform quantization is a well-studied area in signal processing and data compression [S1, S2, unofficial]. Its impact on volume rendering has been carefully studied in the context of efficient encoding and rendering of time-varying volume data [S1, S2, unofficial]. The method is often used as a default in many renderers, although non-uniform and adaptive quantization can more effectively minimize distortion error and enhance data for detecting features [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
