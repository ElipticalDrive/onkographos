# Byte/Short Conversion
Also called: Byte Swap, Short Swap
## Overview
Byte/Short Conversion handles endianness correction for 16-bit voxel buffers and quantizes floating-point or 16-bit integer volume data into 8-bit byte representations for storage or transmission [S1, S2, S3, unofficial]. It operates as a fixed-scale quantization or byte-pair swap step, distinct from adaptive normalization or entropy-coding compression [S1, S2, S3, unofficial].
## How it works
### Inputs and Outputs
- **Float-to-Byte**: Input is a color channel value in $[0.0, 1.0]$. Output is an integer byte in $[0, 255]$ [S2, unofficial].
- **Short-to-Byte**: Input is an array of 16-bit signed shorts. Output is a byte array where each short is decomposed into two bytes [S3, unofficial].
- **Byte-to-Short**: Input is an array of bytes. Output is an array of 16-bit signed shorts [S3, unofficial].
- **Byte Swap**: Input is a 16-bit integer buffer. Output is the same buffer with adjacent byte pairs swapped to correct endianness [S1, unofficial].

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
   d. Return as a tuple of three bytes [S2, unofficial].

#### Short to Byte Conversion (`shortsToBytes`)
1. For each short value $v$ in the input array:
   a. Extract the high byte: $hi = v \gg 8$ (unsigned right shift).
   b. Extract the low byte: $lo = v \ \& \ 0xFF$.
   c. Write $hi$ to the output array.
   d. Write $lo$ to the output array [S3, unofficial].

#### Byte to Short Conversion (`bytesToShorts`)
1. Check if `data.length` is even. If not, throw `IllegalArgumentException` [S3, unofficial].
2. Calculate $n = \text{data.length} / 2$.
3. If `expectedSamples` > 0, check if $n == \text{expectedSamples}$. If not, throw `IllegalStateException` [S3, unofficial].
4. For each pair of bytes $(hi, lo)$ in the input array:
   a. Combine to form short: $v = (hi \ll 8) \ | \ lo$.
   b. Append $v$ to the output short array [S3, unofficial].

#### Byte Swapping for 16-bit Voxel Data
1. Call `LoadFile(..., true)` to trigger byte-pair swapping on the raw 16-bit data buffer before rescaling [S1, unofficial].
2. Iterate over the buffer with stride 2.
3. Swap `pTmp[i]` and `pTmp[i+1]` for each 16-bit value [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def float_to_byte(color_channel: float) -> int:
    """Converts a single float channel [0.0, 1.0] to a byte [0, 255]."""
    clamped = max(0.0, min(1.0, color_channel))
    return round(clamped * 255.0)

def shorts_to_bytes(shorts: list[int]) -> list[int]:
    """Converts a list of shorts to a list of bytes."""
    bytes_out = []
    for v in shorts:
        # Unsigned right shift for 16-bit values
        hi = (v & 0xFFFF) >> 8
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
        v = (hi << 8) | lo
        shorts_out.append(v)
    return shorts_out

def swap_bytes_16bit(buffer: list[int]) -> list[int]:
    """Swaps byte pairs in a 16-bit integer buffer to correct endianness."""
    for i in range(0, len(buffer), 2):
        buffer[i], buffer[i+1] = buffer[i+1], buffer[i]
    return buffer
```

### Parameters
- **Float Scale**: Fixed at $255.0$ [S2, unofficial].
- **Expected Samples**: Optional validation parameter for `bytesToShorts` [S3, unofficial].

### Edge Cases and Pitfalls
- **Float Range**: Values outside $[0.0, 1.0]$ must be clamped to avoid out-of-bounds byte values [S2, unofficial].
- **Byte Array Length**: `bytesToShorts` requires an even number of bytes; an odd length throws an exception [S3, unofficial].
- **Sample Count Mismatch**: If `expectedSamples` is provided and does not match the calculated number of shorts, an exception is thrown [S3, unofficial].
- **Byte Swap Alignment**: The loop condition `i < total * 2 - 2` assumes an even number of voxels; if total is odd, the last byte pair is skipped. Implementation should verify alignment or adjust loop bounds [S1, unofficial].
- **Endianness**: Native little-endian input will not be interpreted correctly unless the swap call or data is adjusted [S1, unofficial].

## Subtopics

### Byte Swapping for 16-bit Voxel Data
Triggers byte-pair swapping on raw 16-bit data buffers via `LoadFile(..., true)` before rescaling [S1, unofficial]. Uses a `Swap(BYTE *p1, BYTE *p2, int nSize)` routine to exchange adjacent bytes [S1, unofficial]. Chosen when file storage endianness mismatches the system architecture. Not suitable for native little-endian input without adjustment [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ for all conversions, where $N$ is the number of elements [G].
- **Memory Complexity**: $O(N)$ for output storage [G].
- **Strengths**: Simple, deterministic, and fast for fixed-range quantization or endianness correction.
- **Limitations**: Lossy for float-to-byte conversion; precision loss depends on original data range. Not given in the sources for measured performance, hardware scaling, or specific resolution benchmarks.

## Background
Originated in graphics and seismic data processing pipelines for efficient storage and transmission of volume data [S1, S3, unofficial]. Implemented in 3D Gaussian Splatting dataset builders for color channel quantization [S2, unofficial] and in seismic data compressors for short/byte serialization [S3, unofficial].

## Sources
[S1] TechCloud-x/IGS, https://github.com/TechCloud-x/IGS, fetched 2026-10-26 (unofficial)
[S2] yumehiko/3dgs-dataset-builder, https://github.com/yumehiko/3dgs-dataset-builder, fetched 2026-10-27 (unofficial)
[S3] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
