# Unquantization Functions

Also called: Unquantize10, Unquantize7, Unquantize9

## Overview
Unquantization functions reconstruct continuous floating-point values from discrete integer indices produced by quantization [S1, unofficial]. They apply specific rounding and scaling constants to map quantized integers back to half-precision or full float ranges, differing from general scalar quantization by using fixed arithmetic offsets [S1, unofficial].

## How it works
**Inputs:** `Int(x)` (quantized integer), `Scale` (normalization divisor) [S1, unofficial].
**Outputs:** `Half(x)` (reconstructed half-precision float) [S1, unofficial].
**Data Structures:** No complex data structures are required; the operation is element-wise [G].
**Steps:**
1. Multiply the input integer `Int(x)` by the scaling factor 65536.
2. Add the rounding bias 32768 to the product.
3. Divide the sum by `Scale` to obtain the final float value.
**Formulas:**
$$ \text{Half}(x) = \frac{\text{Int}(x) \times 65536 + 32768}{\text{Scale}} $$ [S1, unofficial]
Where `Int(x)` is the input integer, 65536 is the scaling multiplier, 32768 is the rounding bias, and `Scale` is the normalization divisor [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unquantize_half(int_val, scale):
    numerator = int_val * 65536 + 32768
    return numerator / scale
```
**Edge Cases and Pitfalls:** Not given in the sources.

## Subtopics

### Finish Unquantize
This variant interpolates between two unquantized endpoints using a weight and converts the result to a final float3 [S1, unofficial].
**Formula:**
$$ \text{Result} = \text{f16tof32}\left( \left( \text{endpoint0Unq} \times (64 - \text{weight}) + \text{endpoint1Unq} \times \text{weight} + 32 \right) \times \frac{31}{4096} \right) $$ [S1, unofficial]
Where `endpoint0Unq` and `endpoint1Unq` are unquantized endpoint values, `weight` is the interpolation weight, 32 is a rounding bias, and 31/4096 is the final scaling factor [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def finish_unquantize(endpoint0_unq, endpoint1_unq, weight):
    interpolated = endpoint0_unq * (64 - weight) + endpoint1_unq * weight + 32
    half_result = interpolated * (31 / 4096)
    return float(half_result)  # f16tof32 conversion [G]
```

## Operating characteristics
Time complexity is O(1) per value [G]. Memory complexity is O(1) auxiliary space [G]. Not given in the sources: measured performance, scaling behavior, or hardware-specific limitations.

## Background
These functions originate from the decoding pipeline of real-time BC6H texture compression [S1, unofficial]. Not given in the sources: specific historical development or broader usage outside this implementation.

## Sources
[S1] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
