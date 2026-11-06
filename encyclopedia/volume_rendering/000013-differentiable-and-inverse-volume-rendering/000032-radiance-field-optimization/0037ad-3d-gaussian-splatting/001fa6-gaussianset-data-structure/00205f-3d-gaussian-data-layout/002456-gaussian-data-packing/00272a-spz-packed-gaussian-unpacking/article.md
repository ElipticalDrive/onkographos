# SPZ Packed Gaussian Unpacking

Also called: Position Unpacking, SPZ Unpacking, unpackPositions

## Overview
SPZ Packed Gaussian Unpacking decodes compressed Gaussian Splat attributes (positions, scales, rotations, alphas, colors, and spherical harmonics) from the SPZ file format into floating-point arrays, handling coordinate system conversion and quantization inversion [S1, unofficial]. It differs from sibling methods like PNG-based compression by using a binary layout with ZSTD compression (v4) or gzip (v1-3) and specific fixed-point/quantized encodings for each attribute type [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A packed attribute buffer (e.g., `packed.positions`), point offset, point count, and a `CoordinateConverter` instance [S1, unofficial].
- Output: A contiguous floating-point array for the unpacked attribute (e.g., `out` for positions) [S1, unofficial].

**Data Structures:**
- `NgspFileHeader` (32 bytes): Contains magic, version, `numPoints`, `shDegree`, `fractionalBits`, `flags`, `numStreams`, and `tocByteOffset` [S1, unofficial].
- `PackedGaussians`: Contains raw byte buffers for `positions`, `scales`, `rotations`, `alphas`, `colors`, and `sh` [S1, unofficial].
- `CoordinateConverter`: Holds flip vectors (`flipP`, `flipQ`, `flipSh`) and optional function pointers (`rotFlipPFunc`, `rotFlipShFuncs`, `rotFlipQFunc`) for coordinate system transformations [S1, unofficial].

**Steps:**
1. **Validation:** Check `pointOffset >= 0`, `pointCount >= 0`, `pointOffset + pointCount <= packed.numPoints`, and `out != nullptr`. If `pointCount == 0`, return true [S1, unofficial].
2. **Buffer Check:** Get buffer for attribute. If buffer empty and `packed.numPoints > 0`, return false [S1, unofficial].
3. **Coordinate Converter:** Create `CoordinateConverter c` [S1, unofficial].
4. **SH Dimension:** Calculate `shDim = dimForDegree(packed.shDegree)` [S1, unofficial].
5. **Dispatch:** Switch on `attr` to call specific unpacker [S1, unofficial].

**Position Unpacking:**
- If `usesFloat16` is true: Cast packed data to `Half*`, iterate `coordCount` (`pointCount * 3`), convert each half to float using `halfToFloat`, and multiply by `c.flipP[i % 3]` [S1, unofficial].
- If `usesFloat16` is false: Calculate `scale = 1.0f / (1 << fractionalBits)`. Iterate `coordCount`, reconstructing a 24-bit signed integer from 3 bytes (little-endian): `fixed32 = src[i*3+0] | (src[i*3+1] << 8) | (src[i*3+2] << 16)` [S1, unofficial]. Sign-extend if bit 23 is set: `fixed32 |= (fixed32 & 0x800000) ? 0xff000000 : 0` [S1, unofficial]. Convert to float: `out[i] = static_cast<float>(fixed32) * scale * c.flipP[i % 3]` [S1, unofficial].
- If `c.rotFlipPFunc` exists, apply it to each point's 3 floats in `out` [S1, unofficial].

**Scale Unpacking:**
- Scales are stored as 8-bit integers per component (x, y, z) [S1, unofficial].
- Decode: `out[i] = src[i] / 16.0f - 10.0f` [S1, unofficial].

**Rotation Unpacking:**
- Two modes based on `usesQuaternionSmallestThree` [S1, unofficial].
- If true: Uses 4 bytes per quaternion (smallest three representation) [S1, unofficial].
- If false: Uses 3 bytes per quaternion (first three components) [S1, unofficial].
- Apply `c.rotFlipQFunc` if present, else apply `c.flipQ` to xyz components [S1, unofficial].

**Alpha Unpacking:**
- Stored as 8-bit integers [S1, unofficial].
- Decode via inverse sigmoid: `out[i] = invSigmoid(src[i] / 255.0f)` [S1, unofficial].

**Color Unpacking:**
- Stored as 3 bytes per point (RGB) [S1, unofficial].
- Decode: `out[i] = ((src[i] / 255.0f) - 0.5f) / colorScale` [S1, unofficial].

**SH Unpacking:**
- Stored as `shDim` coefficients × 3 RGB channels [S1, unofficial].
- **Within-Family:** For each point `p`, for each coefficient `j` in `0..shDim-1`, for each channel `c` in `0..2`: `out[base + j*3 + c] = flip * unquantizeSH(src[base + j*3 + c])` [S1, unofficial].
- **Cross-Family:** If `c.rotFlipShFuncs[0]` exists, for each point `p`, for each band `band` in `0..SH_MAX_DEGREE-1`: if `rotFlipShFuncs[band]` is null, break. Calculate `bandStart = band * (band + 2)` and `bandSize = 2 * band + 3`. If `bandStart + bandSize > shDim`, break. For each channel `0..2`, copy `bandSize` coefficients to `tmp`, apply `rotFlipShFuncs[band](tmp)`, and copy back [S1, unofficial].
- **Pitfall:** If `shDim == 0`, `unpackSh` returns immediately without processing [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dim_for_degree(degree):
    return (degree + 1) ** 2  # [G] Standard SH dimension formula

def half_to_float(h):
    sign = (h >> 15) & 1
    exp = (h >> 10) & 0x1F
    mantissa = h & 0x3FF
    if exp == 0:
        return (-1)**sign * (mantissa / 1024.0)
    elif exp == 31:
        return (-1)**sign * (float('inf') if mantissa else float('nan'))
    else:
        return (-1)**sign * (1.0 + mantissa / 1024.0) * (2 ** (exp - 15))  # [G] IEEE 754 half conversion

def inv_sigmoid(y):
    return math.log(y / (1.0 - y))  # [G] Standard inverse sigmoid

class CoordinateConverter:
    def __init__(self):
        self.flip_p = [1.0, 1.0, 1.0]
        self.flip_q = [1.0, 1.0, 1.0]
        self.flip_sh = [1.0] * 100
        self.rot_flip_p_func = None
        self.rot_flip_q_func = None
        self.rot_flip_sh_funcs = [None] * 10

def unpack_gaussians(packed, point_offset, point_count, coord_converter):
    if point_offset < 0 or point_count < 0 or point_offset + point_count > packed.num_points:
        return False
    if point_count == 0:
        return True
    if packed.num_points > 0 and not packed.positions:
        return False
    
    sh_degree = packed.sh_degree
    sh_dim = dim_for_degree(sh_degree)
    fractional_bits = packed.fractional_bits
    uses_float16 = packed.uses_float16
    uses_smallest_three = packed.uses_quaternion_smallest_three
    color_scale = packed.color_scale  # [G] Parameter from context
    
    cloud = {
        'num_points': point_count,
        'sh_degree': sh_degree,
        'positions': [0.0] * (point_count * 3),
        'scales': [0.0] * (point_count * 3),
        'rotations': [0.0] * (point_count * 4),
        'alphas': [0.0] * point_count,
        'colors': [0.0] * (point_count * 3),
        'sh': [0.0] * (point_count * sh_dim * 3) if sh_dim > 0 else []
    }
    
    # Position Unpacking
    coord_count = point_count * 3
    if uses_float16:
        for i in range(coord_count):
            val = half_to_float(packed.positions[i])
            cloud['positions'][i] = val * coord_converter.flip_p[i % 3]
    else:
        scale = 1.0 / (1 << fractional_bits)
        for i in range(coord_count):
            b0 = packed.positions[i * 3]
            b1 = packed.positions[i * 3 + 1]
            b2 = packed.positions[i * 3 + 2]
            fixed32 = b0 | (b1 << 8) | (b2 << 16)
            if fixed32 & 0x800000:
                fixed32 |= 0xFF000000
            cloud['positions'][i] = float(fixed32) * scale * coord_converter.flip_p[i % 3]
    if coord_converter.rot_flip_p_func:
        for i in range(point_count):
            coord_converter.rot_flip_p_func(cloud['positions'][i*3:i*3+3])
            
    # Scale Unpacking
    for i in range(point_count * 3):
        cloud['scales'][i] = packed.scales[i] / 16.0 - 10.0
        
    # Rotation Unpacking
    for i in range(point_count):
        if uses_smallest_three:
            comp = packed.rotations[i*4] | (packed.rotations[i*4+1] << 8) | (packed.rotations[i*4+2] << 16) | (packed.rotations[i*4+3] << 24)
            i_largest = comp & 0x3
            q = [0.0] * 4
            idx = 0
            for k in range(4):
                if k != i_largest:
                    negbit = (comp >> (9 + idx * 10)) & 1
                    mag = (comp >> (idx * 10)) & 0x1FF
                    val = (mag / 511.0) * math.sqrt(2.0)
                    if negbit:
                        val = -val
                    q[k] = val
                    idx += 1
            sum_sq = sum(v*v for v in q)
            q[i_largest] = math.sqrt(max(0.0, 1.0 - sum_sq))
            norm = math.sqrt(sum(v*v for v in q))
            q = [v / norm for v in q]
        else:
            q = [0.0] * 4
            for k in range(3):
                q[k] = (packed.rotations[i*3 + k] - 127.5) / 127.5 * 2.0
            sum_sq = sum(v*v for v in q[:3])
            q[3] = math.sqrt(max(0.0, 1.0 - sum_sq))
            norm = math.sqrt(sum(v*v for v in q))
            q = [v / norm for v in q]
        if coord_converter.rot_flip_q_func:
            coord_converter.rot_flip_q_func(q)
        else:
            for k in range(3):
                q[k] *= coord_converter.flip_q[k]
        cloud['rotations'][i*4:i*4+4] = q
        
    # Alpha Unpacking
    for i in range(point_count):
        cloud['alphas'][i] = inv_sigmoid(packed.alphas[i] / 255.0)
        
    # Color Unpacking
    for i in range(point_count * 3):
        cloud['colors'][i] = (packed.colors[i] / 255.0 - 0.5) / color_scale
        
    # SH Unpacking
    if sh_dim > 0:
        sh1_bits = 5
        sh_rest_bits = 4
        def unquantize_sh(val, bits):
            max_val = (1 << (bits - 1)) - 1
            return (val - (1 << (bits - 1))) / max_val  # [G] Symmetric quantization
        
        for i in range(point_count):
            base = i * sh_dim * 3
            for j in range(sh_dim):
                for c in range(3):
                    idx = base + j * 3 + c
                    val = packed.sh[idx]
                    bits = sh1_bits if j < dim_for_degree(1) else sh_rest_bits
                    cloud['sh'][idx] = unquantize_sh(val, bits) * coord_converter.flip_sh[j]
                    
        if coord_converter.rot_flip_sh_funcs[0] is not None:
            for i in range(point_count):
                base = i * sh_dim * 3
                for band in range(10):
                    if coord_converter.rot_flip_sh_funcs[band] is None:
                        break
                    band_start = band * (band + 2)
                    band_size = 2 * band + 3
                    if band_start + band_size > sh_dim:
                        break
                    for c in range(3):
                        tmp = []
                        for k in range(band_size):
                            tmp.append(cloud['sh'][base + (band_start + k)*3 + c])
                        coord_converter.rot_flip_sh_funcs[band](tmp)
                        for k in range(band_size):
                            cloud['sh'][base + (band_start + k)*3 + c] = tmp[k]
                            
    return cloud
```

## Operating characteristics
- **Time Complexity:** Linear in the number of points and SH degree. Unpacking is dominated by memory access and simple arithmetic [S1, unofficial].
- **Memory Complexity:** Linear in the number of points and SH degree. Output is $O(N \times (3 + 3 + 4 + 1 + 3 + \text{shDim} \times 3))$ floats [S1, unofficial].
- **Performance:** ZSTD compression (v4) allows parallel decompression of attribute streams, improving throughput on multi-core systems compared to gzip (v1-3) [S1, unofficial].
- **Strengths:** Compact storage, fast unpacking, flexible coordinate system conversion [S1, unofficial].
- **Limitations:** Fixed-point position encoding may lose precision for very large scenes if `fractionalBits` is small [S1, unofficial]. SH rotation requires per-band function calls, which may be slower than matrix multiplication for high degrees [S1, unofficial].

## Background
The SPZ format uses a 32-byte `NgspFileHeader` followed by optional extension records, a table of contents, and independent ZSTD-compressed attribute streams in version 4 [S1, unofficial]. Versions 1-3 used gzip with a single stream [S1, unofficial]. Version 4 improves performance and flexibility by enabling parallel decompression per attribute stream [S1, unofficial].

## Sources
[S1] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
