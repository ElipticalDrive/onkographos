# 3D Volume Block Compression (Closed-Loop)

Also called: Closed-Loop DPCM Compression, TraceBlockCodec.compress, closed-loop 3D DPCM, closed-loop quantization

## Overview
This method compresses 3D seismic data blocks by normalizing samples to a fixed range, applying closed-loop differential pulse-code modulation (DPCM) with quantization, and packing the result with Deflate compression [S1, unofficial]. It differs from open-loop DPCM by feeding the reconstructed quantized value back into the predictor, ensuring the decoder reproduces the exact quantization steps without requiring the original float values during decoding [S1, unofficial]. It builds on the 3D Seismic Data Compression Pipeline [11727-3d-seismic-data-compression-pipeline.md].

## How it works
**Inputs:**
- `data`: A 3D array of floating-point samples with dimensions $n_i \times n_j \times n_t$ [S1, unofficial].
- `profile`: An object providing `effectiveBits()` (quantization bit depth) and `deflaterLevel()` (Deflate compression level) [S1, unofficial].

**Outputs:**
- `CompressedVolumeBlock3D`: An object containing `metadata` (original block metadata), `min` (minimum original value), `max` (maximum original value), and `payload` (compressed byte array) [S1, unofficial].

**Data Structures:**
- `q`: An integer array of length $n_i \times n_j \times n_t$, storing quantized residual values [S1, unofficial].
- `norm`: A 3D array of floats in the range $[-1, 1]$, storing normalized data [S1, unofficial].
- `payload`: A byte array resulting from packing `q` and applying Deflate [S1, unofficial].

**Parameters:**
- `bits`: The number of bits used for quantization, obtained from `profile.effectiveBits()` [S1, unofficial].
- `maxQ`: The maximum quantization level, calculated as $2^{\text{bits}} - 1$ [G].
- `inv`: The inverse of `maxQ`, calculated as $1.0 / \text{maxQ}$ [G].
- `deflaterLevel`: The compression level for Deflate, obtained from `profile.deflaterLevel()` [S1, unofficial].

**Steps:**

1. **Compute Min/Max:** Iterate over all indices $i, j, k$ to find the minimum ($min$) and maximum ($max$) values in `data` [S1, unofficial].
2. **Normalize:** Normalize `data` to the range $[-1, 1]$ to produce `norm` [S1, unofficial]:
   - Calculate range: $range = max - min$ [S1, unofficial].
   - If $range == 0$, set all `norm[i][j][k] = 0` [S1, unofficial].
   - Otherwise, for each sample $v$ in `data`:
     - $zeroToOne = (v - min) / range$ [S1, unofficial].
     - $norm[i][j][k] = zeroToOne \times 2 - 1$ [S1, unofficial].
3. **Closed-Loop Quantization:** Encode `norm` into integer array `q` using closed-loop DPCM [S1, unofficial]:
   - Initialize `prev = 0.0` [S1, unofficial].
   - For each sample index $idx$ from $0$ to $n_i \times n_j \times n_t - 1$:
     - Let $current = norm[idx]$ (linearized) [S1, unofficial].
     - Calculate residual: $residual = current - prev$ [S1, unofficial].
     - Quantize residual: $q[idx] = \text{quantize}(residual, maxQ)$ [S1, unofficial].
     - Reconstruct previous value: $prev = \text{reconstruct}(prev, q[idx], inv)$ [S1, unofficial].
4. **Pack and Compress:** Pack the integer array `q` into a byte array, storing `bits` in the payload header, and apply Deflate compression with `deflaterLevel` [S1, unofficial].
5. **Return:** Construct and return the `CompressedVolumeBlock3D` object with `metadata`, `min`, `max`, and the compressed `payload` [S1, unofficial].

**Decompression:**

1. **Unpack:** Unpack the `payload` to retrieve the integer array `q` and extract `bits` from the header [S1, unofficial].
2. **Decode:** Decode `q` into the normalized float array `norm` using closed-loop DPCM reversal [S1, unofficial].
3. **Denormalize:** Convert `norm` back to the original scale to produce `data` [S1, unofficial]:
   - Calculate range: $range = max - min$ [S1, unofficial].
   - If $range == 0$, set all `data[i][j][k] = min` [S1, unofficial].
   - Otherwise, for each normalized value $v$ in `norm`:
     - $zeroToOne = (v + 1) / 2$ [S1, unofficial].
     - $data[i][j][k] = min + zeroToOne \times range$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import zlib
from dataclasses import dataclass
import math

@dataclass
class CompressedVolumeBlock3D:
    metadata: object
    min: float
    max: float
    payload: bytes

def quantize(residual: float, max_q: int) -> int:
    # [G] Standard symmetric quantization: round to nearest integer in [-max_q, max_q]
    val = int(round(residual * max_q))
    return max(-max_q, min(max_q, val))

def reconstruct(prev: float, q_val: int, inv: float) -> float:
    # [G] Standard closed-loop reconstruction: add scaled quantized residual
    return prev + q_val * inv

def pack_bits(q: list, bits: int, level: int) -> bytes:
    # [G] Pack integers into bytes, prepend 4-byte header with bit depth, then Deflate
    header = bits.to_bytes(4, byteorder='little', signed=True)
    packed = bytearray()
    for val in q:
        for b in range(bits):
            packed.append((val >> b) & 1)
    # Pad to full byte
    while len(packed) % 8 != 0:
        packed.append(0)
    # Convert bits to bytes
    byte_data = bytearray()
    for i in range(0, len(packed), 8):
        byte_val = 0
        for j in range(8):
            if i + j < len(packed):
                byte_val |= packed[i + j] << j
        byte_data.append(byte_val)
    return header + zlib.compress(bytes(byte_data), level)

def unpack_bits(payload: bytes, count: int, bits: int) -> list:
    # [G] Reverse pack_bits: decompress, strip header, unpack bits to integers
    raw = zlib.decompress(payload[4:])
    bits_list = []
    for byte in raw:
        for b in range(8):
            bits_list.append((byte >> b) & 1)
    q = []
    for idx in range(count):
        val = 0
        for b in range(bits):
            if idx * bits + b < len(bits_list):
                val |= bits_list[idx * bits + b] << b
        # Sign extend if necessary (assuming symmetric range)
        if val >= (1 << (bits - 1)):
            val -= (1 << bits)
        q.append(val)
    return q

def compress(data, ni, nj, nt, profile):
    # Step 1: Compute Min/Max
    min_val = data[0][0][0]
    max_val = data[0][0][0]
    for i in range(ni):
        for j in range(nj):
            for k in range(nt):
                if data[i][j][k] < min_val:
                    min_val = data[i][j][k]
                if data[i][j][k] > max_val:
                    max_val = data[i][j][k]

    # Step 2: Normalize
    range_val = max_val - min_val
    norm = [[[0.0 for _ in range(nt)] for _ in range(nj)] for _ in range(ni)]
    if range_val == 0:
        for i in range(ni):
            for j in range(nj):
                for k in range(nt):
                    norm[i][j][k] = 0.0
    else:
        for i in range(ni):
            for j in range(nj):
                for k in range(nt):
                    v = data[i][j][k]
                    zero_to_one = (v - min_val) / range_val
                    norm[i][j][k] = zero_to_one * 2.0 - 1.0

    # Step 3: Closed-Loop Quantization
    bits = profile.effectiveBits()
    max_q = (1 << bits) - 1
    inv = 1.0 / max_q
    q = [0] * (ni * nj * nt)
    prev = 0.0
    
    for idx in range(ni * nj * nt):
        i, rem = divmod(idx, nj * nt)
        j, k = divmod(rem, nt)
        current = norm[i][j][k]
        residual = current - prev
        q[idx] = quantize(residual, max_q)
        prev = reconstruct(prev, q[idx], inv)

    # Step 4: Pack and Compress
    payload = pack_bits(q, bits, profile.deflaterLevel())

    # Step 5: Return
    return CompressedVolumeBlock3D(metadata=None, min=min_val, max=max_val, payload=payload)

def decompress(compressed_block, ni, nj, nt):
    # Step 1: Unpack
    bits = int.from_bytes(compressed_block.payload[:4], byteorder='little', signed=True)
    q = unpack_bits(compressed_block.payload, ni * nj * nt, bits)

    # Step 2: Decode
    norm_flat = [0.0] * (ni * nj * nt)
    prev = 0.0
    inv = 1.0 / ((1 << bits) - 1)
    for idx in range(ni * nj * nt):
        norm_flat[idx] = prev
        prev = reconstruct(prev, q[idx], inv)

    # Step 3: Denormalize
    min_val = compressed_block.min
    max_val = compressed_block.max
    range_val = max_val - min_val
    data = [[[0.0 for _ in range(nt)] for _ in range(nj)] for _ in range(ni)]

    if range_val == 0:
        for i in range(ni):
            for j in range(nj):
                for k in range(nt):
                    data[i][j][k] = min_val
    else:
        idx = 0
        for i in range(ni):
            for j in range(nj):
                for k in range(nt):
                    v = norm_flat[idx]
                    zero_to_one = (v + 1.0) / 2.0
                    data[i][j][k] = min_val + zero_to_one * range_val
                    idx += 1

    return data
```

**Edge Cases and Pitfalls:**
- **Zero Range:** If $max == min$, normalization results in all zeros, and denormalization must fill the output with $min$ [S1, unofficial].
- **Precision:** Closed-loop quantization ensures decoder consistency but introduces quantization error, trading file size for accuracy based on `effectiveBits` [S1, unofficial].
- **Ordering:** The linearization of the 3D array for DPCM must be consistent between compression and decompression [G].

## Subtopics

### Open-Loop DPCM
Uses the original float values for prediction instead of the reconstructed quantized values [G].

### TraceBlockCodec.inflate
Legacy decompression path that inflates bytes to shorts and decodes residuals linearly [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples $N = n_i \times n_j \times n_t$ for normalization, quantization, and denormalization [G].
- **Memory Complexity:** Requires storage for the original data, normalized data, quantized integer array, and compressed payload [G].
- **Performance:** Higher bit depths result in larger file sizes but better accuracy, while Deflate level affects compression ratio and CPU time [S1, unofficial].
- **Strengths:** Closed-loop design ensures decoder stability, and normalization allows efficient quantization across varying data ranges [G].
- **Limitations:** Lossy compression; performance is sensitive to the choice of bit depth and Deflate level [S1, unofficial].

## Background
This method originates from the SeismicDataCompressor repository [S1, unofficial]. It is designed to integrate with the 3D Seismic Data Compression Pipeline, providing a robust way to compress 3D volume blocks while preserving trace integrity [11727-3d-seismic-data-compression-pipeline.md].

## Sources
[S1] SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
