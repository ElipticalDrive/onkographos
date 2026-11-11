# Color Packing (RGBA)


## Overview
Color Packing (RGBA) is a lossless compression method that maps 4-component floating-point color vectors into a compact 4-byte unsigned integer representation with 8-bit precision per channel. It solves the problem of reducing memory bandwidth and storage requirements for volumetric color data by discarding floating-point precision in favor of hardware-friendly integer formats. It differs from [16-bit Floating Point Format for Spectra](1163-16-bit-floating-point-format-for-spectra.md) by using fixed-point integer storage rather than half-precision floats, and from [ELSCINT1/PMSCT_RLE1 Delta Decoding](11897-elscint1-pmsct-rle1-delta-decoding.md) by operating on individual color components rather than applying run-length encoding to delta-encoded spectral data.

## How it works
**Inputs:** A 4-component floating-point color vector $\mathbf{c} = [c_0, c_1, c_2, c_3]$ where each component is a `float`.
**Outputs:** A 4-byte unsigned integer array (or a single 32-bit unsigned integer) representing the packed color.

**Data Structures:**
The output is a 32-bit value composed of four 8-bit fields.
- Byte 0: Red ($c_0$)
- Byte 1: Green ($c_1$)
- Byte 2: Blue ($c_2$)
- Byte 3: Alpha ($c_3$)

**Steps:**
1. For each component $c_i$ where $i \in \{0, 1, 2, 3\}$:
   a. Multiply the component value by 255.0.
   b. Cast the result to an 8-bit unsigned integer (`u8`). This truncates any fractional part.
   c. Store the result in the $i$-th byte of the output.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_color_rgba(c: list[float]) -> int:
    """
    Packs a 4-component float color vector into a 32-bit unsigned integer.
    
    Args:
        c: List of 4 floats [R, G, B, A].
        
    Returns:
        32-bit integer with R in bits 0-7, G in 8-15, B in 16-23, A in 24-31.
    """
    # Step 1: Quantize each component to 8-bit unsigned integer
    r = int(c[0] * 255.0) & 0xFF
    g = int(c[1] * 255.0) & 0xFF
    b = int(c[2] * 255.0) & 0xFF
    a = int(c[3] * 255.0) & 0xFF
    
    # Step 2: Pack into a single 32-bit integer
    packed = r | (g << 8) | (b << 16) | (a << 24)
    
    return packed
```

**Parameters:**
- Scale factor: 255.0.
- Bit depth: 8 bits per channel.

**Edge Cases and Pitfalls:**
- **Precision Loss:** Casting to `u8` discards sub-byte precision. Values between 0 and $1/255$ all map to 0.
- **Clamping:** The sources do not explicitly specify clamping of input values to $[0, 1]$ before multiplication. If input values exceed 1.0, the result may overflow the 8-bit range before masking. The bitwise AND with `0xFF` acts as a wrap-around modulo 256, which may produce incorrect colors for out-of-range inputs. [G] In practice, inputs should be clamped to $[0, 1]$ to ensure correct mapping, but the source definition does not mandate this step.
- **Negative Values:** Negative inputs will result in negative integers before masking, which wrap around to high positive values (e.g., -1 becomes 255). This is likely unintended for color data.

## Subtopics

### packDCColor
Packs the DC spherical harmonic coefficient (RGB) and alpha into a single 32-bit unsigned integer for GPU texture storage. It is used when storing the zeroth-order spherical harmonic coefficients, which represent the average color of a volume element. It differs from standard RGBA packing by applying a specific quantization constant derived from the SH basis function $Y_0^0$. Choose this when storing SH-based volumetric representations that require GPU texture compatibility. [S2, unofficial]

**Steps:**
1. Define quantization lambda $k_{SH0} = 0.28209479177387814$ (the value of $Y_0^0$).
2. For each color component $sh[j]$ where $j \in \{0, 1, 2\}$:
   a. Compute $val = k_{SH0} \times sh[j]$.
   b. Add 0.5 to $val$.
   c. Clamp $val$ to the range $[0, 1]$.
   d. Multiply by 255.
   e. Cast to `uint32` and mask with `0xFF` to get the 8-bit value.
3. For the alpha component $\alpha$:
   a. Clamp $\alpha$ to $[0, 1]$.
   b. Multiply by 255.
   c. Cast to `uint32` and mask with `0xFF`.
4. Pack the results: $R | (G \ll 8) | (B \ll 16) | (A \ll 24)$.

**Formula:**
$$ R = \text{quant}(k_{SH0} \times sh[0] + 0.5) $$
$$ G = \text{quant}(k_{SH0} \times sh[1] + 0.5) $$
$$ B = \text{quant}(k_{SH0} \times sh[2] + 0.5) $$
$$ A = \text{quant}(\alpha + 0.5) $$
where $\text{quant}(x) = \text{clamp}(x, 0, 1) \times 255$ cast to `uint8`.

**Parameter:**
- $k_{SH0} = 0.28209479177387814$ [S2, unofficial]

### RGBE HDR Encoding
Packs high-dynamic-range (HDR) color values into a 4-byte format using a shared exponent for the RGB channels. It is used to store HDR volumetric colors in a compact format that preserves dynamic range. It differs from standard RGBA packing by using a variable exponent to extend the range of representable values beyond $[0, 1]$. Choose this when storing HDR volumetric data where color intensities can exceed 1.0. [S3, unofficial]

**Steps:**
1. Compute $d = \max(c.x, c.y, c.z)$.
2. If $d \le 10^{-32}$, return `HDRPixel(0, 0, 0, 0)`.
3. Extract exponent $e$ and mantissa $m$ from $d$ using `frexp(d, &e)`, such that $d = m \times 2^e$.
4. Compute scale factor $s = m \times 256.0 / d$.
5. Compute packed components:
   - $r = \text{uint8\_t}(c.x \times s)$
   - $g = \text{uint8\_t}(c.y \times s)$
   - $b = \text{uint8\_t}(c.z \times s)$
   - $e_{packed} = \text{uint8\_t}(e + 128)$
6. Return `HDRPixel` with fields $r, g, b, e_{packed}$.

**Layout:**
`HDRPixel`: 4-byte struct with fields `uint8_t r, g, b, e` in that order.

**Parameters:**
- Threshold: $10^{-32}$ for near-zero values.
- Exponent bias: 128.

**Pitfalls:**
- Channel values $c.x, c.y, c.z$ can exceed 1.0 after scaling, causing overflow when cast to `uint8_t`. The code does not clamp them, relying on the HDR format's ability to store values >1 via the exponent. [S3, unofficial]

## Operating characteristics
**Time Complexity:** $O(1)$ per color vector. The operation involves a constant number of arithmetic operations and type casts.
**Memory Complexity:** Reduces storage from 16 bytes (4 floats) to 4 bytes (1 uint32), achieving a 4:1 compression ratio.
**Measured Performance:** Not given in the sources.
**Strengths:** Extremely fast, hardware-friendly, and widely supported by graphics APIs.
**Limitations:** Loss of precision, limited dynamic range (unless using variants like RGBE), and potential for overflow if inputs are not clamped.
**When to use:** When memory bandwidth is a bottleneck and color precision requirements are low (e.g., display-space colors).
**When not to use:** When high dynamic range is required (use RGBE), when spectral data is stored (use 16-bit FP), or when delta-encoded data is more efficient (use RLE).

## Background
Color Packing (RGBA) is a fundamental technique in computer graphics for storing color data in compact formats. It is commonly used in texture storage, framebuffer formats, and volumetric rendering pipelines where memory efficiency is critical. The method is derived from the standard RGBA color model and is supported by virtually all graphics hardware.

## Sources
[S1] B4rtekk1/Minerust, https://github.com/B4rtekk1/Minerust, fetched 2026-10-27 (unofficial)
[S2] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S3] RNG65536/CUDA-volpath, https://github.com/RNG65536/CUDA-volpath, fetched 2026-10-26 (unofficial)
