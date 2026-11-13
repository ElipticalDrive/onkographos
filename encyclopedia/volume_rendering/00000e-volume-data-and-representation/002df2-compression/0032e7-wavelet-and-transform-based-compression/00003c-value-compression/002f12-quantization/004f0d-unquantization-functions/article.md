# Unquantization Functions

Also called: Unquantize10, Unquantize7, Unquantize9

## Overview
Unquantization functions reconstruct continuous or high-precision floating-point values from discrete, integer-quantized representations. In the context of [Wavelet and Transform-based Compression](52-wavelet-and-transform-based-compression.md), specifically [Value compression](53-value-compression.md) using [Quantization](5935-quantization.md), these functions serve as the inverse of the quantization step. Unlike [Scalar Quantization](5726-scalar-quantization.md) which maps uniformly, these specific implementations handle the non-linear or block-based reconstruction required by formats like BC6H, converting stored integer codes back to half-precision or full float ranges.

## How it works
The method operates on integer codes produced by the quantization stage. It supports two primary modes: basic half-precision reconstruction and weighted interpolation for higher precision.

### Inputs and Outputs
- **Input**: Integer quantized value(s) $I$, scale factor $S$, and optionally a weight $W$ and endpoint values.
- **Output**: Reconstructed floating-point value(s) $\hat{x}$.

### Data Structures
The implementation typically processes blocks of data. For the "Finish Unquantize" variant, inputs include two unquantized endpoints ($E_0, E_1$) and a weight $W$.

### Formulas and Steps

#### Basic Unquantization (Half Precision)
Reconstructs a half-precision float from an integer code.

1.  **Scale and Offset**: Multiply the integer code $I$ by the scale factor $S$. Add an offset of 32768 to center the rounding.
2.  **Division**: Divide the result by the scale factor $S$.
3.  **Conversion**: Convert the resulting integer to a half-precision float.

$$ \text{Half}(I) = \frac{I \cdot 65536 + 32768}{S} $$

Where:
- $I$: The integer quantized value.
- $S$: The scale factor used during quantization.
- The constant 65536 and 32768 are derived from the bit-depth and rounding requirements for half-precision conversion.

#### Finish Unquantization (Interpolated)
Reconstructs a final float3 value by interpolating between two unquantized endpoints using a weight.

1.  **Weighted Sum**: Compute the weighted sum of the two unquantized endpoints $E_0$ and $E_1$ using weight $W$.
    $$ \text{Sum} = E_0 \cdot (64 - W) + E_1 \cdot W $$
2.  **Rounding Offset**: Add 32 to the sum for rounding.
3.  **Scaling**: Multiply by the factor $\frac{31}{4096}$.
4.  **Conversion**: Convert the result to a full 32-bit float.

$$ \text{Result} = \text{f16tof32}\left( \left( E_0 \cdot (64 - W) + E_1 \cdot W + 32 \right) \cdot \frac{31}{4096} \right) $$

Where:
- $E_0, E_1$: The unquantized endpoint values (typically half-precision).
- $W$: The interpolation weight (integer, range 0-63).
- $\text{f16tof32}$: Conversion from half-precision to single-precision float.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unquantize_half(int_val: int, scale: float) -> float:
    """
    Reconstructs a half-precision float from an integer code.
    """
    # Formula: (Int(x) * 65536 + 32768) / Scale
    numerator = int_val * 65536 + 32768
    result = numerator / scale
    return result

def finish_unquantize(endpoint0: float, endpoint1: float, weight: int) -> float:
    """
    Interpolates between two unquantized endpoints using a weight and converts to final float3.
    """
    # Formula: f16tof32( (endpoint0Unq * (64 - weight) + endpoint1Unq * weight + 32) * (31 / 4096) )
    
    # Calculate weighted sum
    weighted_sum = endpoint0 * (64 - weight) + endpoint1 * weight
    
    # Add rounding offset
    rounded_sum = weighted_sum + 32
    
    # Apply scaling factor
    scaled_value = rounded_sum * (31 / 4096)
    
    # Convert to float32 (simulated)
    result = float(scaled_value)
    return result
```

### Edge Cases and Pitfalls
- **Division by Zero**: Ensure the scale factor $S$ is non-zero.
- **Precision Loss**: The conversion from half to float may introduce minor precision artifacts if not handled correctly.
- **Weight Range**: The weight $W$ is typically constrained to an integer range (e.g., 0-63). Values outside this range may produce incorrect interpolation results.

## Subtopics

### Finish Unquantize
Interpolates between two unquantized endpoints using a weight and converts to final float3. It is used when higher precision is required than what single endpoints provide. Choose this when the compression format uses weighted interpolation between endpoints (e.g., BC6H). Link: [Finish Unquantize](#finish-unquantize).

## Operating characteristics
- **Time Complexity**: $O(1)$ per value.
- **Memory Complexity**: $O(1)$ auxiliary space.
- **Performance**: Very fast, suitable for real-time GPU decompression.
- **Strengths**: Efficient reconstruction of compressed data.
- **Limitations**: Precision is limited by the original quantization step.

## Background
Originating from real-time BC6H compression implementations, these functions are part of the decompression pipeline for block-compressed textures. They address the need to efficiently convert stored integer codes back to usable floating-point values for rendering.

## Sources
[S1] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
