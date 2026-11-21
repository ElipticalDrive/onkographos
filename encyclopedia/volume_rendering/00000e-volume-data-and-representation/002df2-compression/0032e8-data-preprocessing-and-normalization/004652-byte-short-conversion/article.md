# Byte/Short Conversion
Also called: Color Conversion, Float to Byte Conversion, bytesToShorts, shortsToBytes
## Overview
Byte/Short Conversion maps floating-point volume data or color channels to integer representations (8-bit bytes or 16-bit shorts) for storage or transmission efficiency [S1, S2, unofficial]. It differs from general normalization by performing a fixed-scale quantization without adaptive histogram stretching, and from compression by being a lossy encoding step rather than a entropy-coding or transform step [S1, S2, unofficial].
## How it works
### Inputs and Outputs
- **Float-to-Byte**: Input is a color channel value in the range $[0.0, 1.0]$. Output is an integer byte in $[0, 255]$.
- **Short-to-Byte**: Input is an array of 16-bit signed shorts. Output is a byte array where each short is decomposed into two bytes.
- **Byte-to-Short**: Input is an array of bytes. Output is an array of 16-bit signed shorts.

### Data Structures
- **Byte**: 8-bit unsigned integer.
- **Short**: 16-bit signed integer.
- **Byte Array**: Sequential storage of bytes.
- **Short Array**: Sequential storage of shorts.

### Steps

#### Float to Byte Conversion
1. For each of the first three channels in the input color:
   a. Clamp the value to $[0.0, 1.0]$.
   b. Multiply by $255.0$.
   c. Round to the nearest integer.
   d. Return as a tuple of three bytes [S1, unofficial].

#### Short to Byte Conversion (`shortsToBytes`)
1. For each short value $v$ in the input array:
   a. Extract the high byte: $hi = v \gg 8$ (unsigned right shift).
   b. Extract the low byte: $lo = v \ \& \ 0xFF$.
   c. Write $hi$ to the output array.
   d. Write $lo$ to the output array [S2, unofficial].

#### Byte to Short Conversion (`bytesToShorts`)
1. Check if `data.length` is even. If not, throw `IllegalArgumentException` [S2, unofficial].
2. Calculate $n = \text{data.length} / 2$.
3. If `expectedSamples` > 0, check if $n == \text{expectedSamples}$. If not, throw `IllegalStateException` [S2, unofficial].
4. For each pair of bytes $(hi, lo)$ in the input array:
   a. Combine to form short: $v = (hi \ll 8) \ | \ lo$.
   b. Append $v$ to the output short array [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def float_to_byte(color_channel: float) -> int:
    """Converts a single float channel [0.0, 1.0] to a byte [0, 255]."""
    # Clamp to [0.0, 1.0]
    clamped = max(0.0, min(1.0, color_channel))
    # Multiply by 255 and round to nearest integer
    return round(clamped * 255.0)

def shorts_to_bytes(shorts: list[int]) -> list[int]:
    """Converts a list of shorts to a list of bytes."""
    bytes_out = []
    for v in shorts:
        # High byte: unsigned right shift by 8
        hi = v >> 8
        # Low byte: mask with 0xFF
        lo = v & 0xFF
        bytes_out.append(hi)
        bytes_out.append(lo)
    return bytes_out

def bytes_to_shorts(data: list[int], expected_samples: int = -1) -> list[int]:
    """Converts a list of bytes to a list of shorts."""
    if len(data) % 2 != 0:
        raise ValueError("data.length is odd")
    
    n = len(data) // 2
    
    if expected_samples > 0 and n != expected_samples:
        raise ValueError(f"n ({n}) != expectedSamples ({expected_samples})")
        
    shorts_out = []
    for i in range(0, len(data), 2):
        hi = data[i]
        lo = data[i+1]
        # Combine: (hi << 8) | lo
        v = (hi << 8) | lo
        shorts_out.append(v)
    return shorts_out
```

### Parameters
- **Float Scale**: Fixed at 255.0 [S1, unofficial].
- **Expected Samples**: Optional parameter for validation in `bytesToShorts` [S2, unofficial].

### Edge Cases and Pitfalls
- **Float Range**: Values outside $[0.0, 1.0]$ must be clamped to avoid out-of-bounds byte values [S1, unofficial].
- **Byte Array Length**: `bytesToShorts` requires an even number of bytes; an odd length throws an exception [S2, unofficial].
- **Sample Count Mismatch**: If `expectedSamples` is provided and does not match the calculated number of shorts, an exception is thrown [S2, unofficial].
- **Sign Extension**: When converting shorts to bytes, ensure the high byte extraction handles signed shorts correctly (unsigned shift) [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ for all conversions, where $N$ is the number of elements.
- **Memory Complexity**: $O(N)$ for output storage.
- **Strengths**: Simple, fast, deterministic.
- **Limitations**: Lossy; precision loss depends on the original data range and bit depth. Not suitable for high-precision scientific data without careful scaling [S1, S2, unofficial].

## Background
Originated in graphics and seismic data processing pipelines for efficient storage and transmission of volume data [S1, S2, unofficial]. Commonly used in 3D Gaussian Splatting dataset builders and seismic data compressors [S1, S2, unofficial].

## Sources
[S1] yumehiko/3dgs-dataset-builder, https://github.com/yumehiko/3dgs-dataset-builder, fetched 2026-10-27 (unofficial)
[S2] https://github.com/rubensrudio/SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
