# Unquantization Functions

Also called: Unquantize10, Unquantize7, Unquantize9

## Overview
Unquantization functions reconstruct continuous floating-point values from discrete integer codes produced by [Quantization](5935-quantization.md). In the context of BC6H compression, these functions map 10-bit or 7-bit integer indices back to half-precision or full-precision float ranges. They differ from sibling methods by their specific fixed-point arithmetic implementations designed for hardware acceleration in block compression, prioritizing low-latency reconstruction over general-purpose precision.

## How it works
**Inputs:** Integer quantized values (indices), scaling factors, and interpolation weights.
**Outputs:** Reconstructed floating-point values (half or full precision).

**Data Structures:**
- `Int(x)`: The integer code (e.g., 10-bit or 7-bit unsigned integer).
- `Scale`: A normalization factor derived from the quantization step.
- `weight`: An interpolation factor between two endpoints (0 to 63 for 6-bit weight).

**Formulas:**

1.  **Half Precision Unquantization (Unquantize10/9):**
    Converts an integer code to a half-precision float.
    $$ \text{Half}(x) = \frac{\text{Int}(x) \times 65536 + 32768}{\text{Scale}} $$
    Where:
    - $\text{Int}(x)$ is the input integer value.
    - $65536$ is $2^{16}$, used for fixed-point scaling.
    - $32768$ is $2^{15}$, used for rounding (bias).
    - $\text{Scale}$ is the divisor determined by the quantization range.

2.  **Finish Unquantization (Interpolation):**
    Interpolates between two unquantized endpoints using a weight to produce a final float3 result.
    $$ \text{Result} = \text{f16tof32}\left( \frac{(\text{endpoint0Unq} \times (64 - \text{weight}) + \text{endpoint1Unq} \times \text{weight} + 32) \times 31}{4096} \right) $$
    Where:
    - $\text{endpoint0Unq}$ and $\text{endpoint1Unq}$ are the unquantized half-precision values from the first step.
    - $\text{weight}$ is an integer in $[0, 63]$.
    - $32$ is the rounding bias for the 6-bit weight interpolation.
    - $31/4096$ is the scaling factor to map the interpolated integer result back to the float range.
    - $\text{f16tof32}$ converts the half-precision intermediate result to a full 32-bit float.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unquantize_half(int_val: int, scale: float) -> float:
    """
    Converts an integer quantized value to half-precision float range.
    Formula: (Int(x) * 65536 + 32768) / Scale
    """
    # Fixed-point calculation with rounding bias
    numerator = (int_val * 65536) + 32768
    return numerator / scale

def finish_unquantize(endpoint0_unq: float, endpoint1_unq: float, weight: int) -> float:
    """
    Interpolates between two unquantized endpoints using a weight.
    Formula: f16tof32( (endpoint0Unq * (64 - weight) + endpoint1Unq * weight + 32) * (31 / 4096) )
    """
    # Calculate weighted sum with rounding bias
    # Note: endpoint values are assumed to be in half-precision range or scaled integers
    # The formula implies integer arithmetic on scaled values before final conversion
    weighted_sum = (endpoint0_unq * (64 - weight)) + (endpoint1_unq * weight) + 32
    scaled_result = weighted_sum * (31 / 4096)
    
    # Convert to full float32
    # In implementation, this step handles the final precision conversion
    return float(scaled_result)
```

**Parameters:**
- `scale`: Typically derived from the maximum representable value in the quantization range. Not given in the sources.
- `weight`: Integer in range $[0, 63]$.

**Edge Cases:**
- Division by zero if `Scale` is 0.
- Overflow if `Int(x)` is large and `Scale` is small.
- Precision loss during `f16tof32` conversion if not handled correctly.

## Subtopics

### Finish Unquantize
Interpolates between two unquantized endpoints using a weight and converts to final float3. It is used when a block requires more than two representative values, allowing for finer granularity in color representation. Choose this when higher visual quality is needed at the cost of additional computation. Link to [Quantization](5935-quantization.md).

## Operating characteristics
- **Time Complexity:** O(1) per pixel/voxel.
- **Memory Complexity:** O(1).
- **Performance:** Highly optimized for GPU hardware due to fixed-point arithmetic.
- **Strengths:** Low latency, hardware-friendly.
- **Limitations:** Limited precision compared to full float32.
- **When to use:** Real-time rendering with compressed textures.
- **When not to use:** High-precision scientific visualization.

## Background
Originated in the context of BC6H texture compression, developed for efficient real-time rendering. The method is part of the broader family of transform-based compression techniques.

## Sources
[S1] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
