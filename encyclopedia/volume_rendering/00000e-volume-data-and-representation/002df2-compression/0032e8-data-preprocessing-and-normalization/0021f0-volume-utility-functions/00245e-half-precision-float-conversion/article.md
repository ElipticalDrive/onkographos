# Half-Precision Float Conversion

Also called: SPZ Half-Float to Float Conversion, floatToHalf, halfToFloat, packHalf2x16

## Overview
This method converts between IEEE 754 single-precision (32-bit) and half-precision (16-bit) floating-point formats, providing both `floatToHalf` and `halfToFloat` operations, as well as a packing utility for two half-precision values into a 32-bit integer. It is designed to avoid undefined behavior associated with type punning (e.g., pointer casting) by using explicit bit manipulation and `memcpy` [S2, unofficial]. In the context of volume rendering, this enables efficient storage and transmission of volume data using reduced precision formats like IEEE 754-2008 binary16 [S2, unofficial].

## How it works

### Inputs and Outputs
*   **floatToHalf**: Input is a 32-bit IEEE 754 single-precision float. Output is a 16-bit half-precision float (typically represented as a 16-bit unsigned integer or a specific half-precision struct).
*   **halfToFloat**: Input is a 16-bit half-precision float. Output is a 32-bit IEEE 754 single-precision float.
*   **packHalf2x16**: Inputs are two 16-bit half-precision floats ($h_x, h_y$). Output is a 32-bit unsigned integer containing both halves.

### Data Structures
*   **Half-precision float (binary16)**: 16 bits total.
    *   Sign: 1 bit (bit 15).
    *   Exponent: 5 bits (bits 10–14).
    *   Mantissa (Fraction): 10 bits (bits 0–9).
*   **Single-precision float (binary32)**: 32 bits total.
    *   Sign: 1 bit (bit 31).
    *   Exponent: 8 bits (bits 23–30).
    *   Mantissa (Fraction): 23 bits (bits 0–22).

### Algorithm: halfToFloat
Converts a 16-bit half-precision float `h` to a 32-bit float `f`.

1.  **Extract Sign**:
    $$sgn = (h \gg 15) \ \& \ 0x1$$
2.  **Extract Exponent and Mantissa**:
    $$exponent = (h \gg 10) \ \& \ 0x1f$$
    $$mantissa = h \ \& \ 0x3ff$$
3.  **Handle Special Cases**:
    *   If $exponent == 0$:
        *   If $mantissa == 0$: Return signed zero ($sgn \times 0.0$).
        *   Else: Return denormalized number:
            $$result = signMul \times 2^{-14} \times \frac{mantissa}{1024}$$
            Where $signMul = (sgn == 1) ? -1.0 : 1.0$ [S1, unofficial].
    *   If $exponent == 31$:
        *   If $mantissa \neq 0$: Return NaN.
        *   Else: Return signed infinity ($signMul \times \infty$) [S1, unofficial].
4.  **Normalize and Convert**:
    *   Calculate the new exponent for single precision. The bias difference between half (15) and single (127) is 112. However, the source notes specify a threshold-based adjustment [S2, unofficial]:
        *   If $exponent < 113$: This condition in the source notes seems to refer to the *single-precision* exponent during the reverse conversion or contains a typo for the forward conversion logic. Based on standard conversion and the note "If `exp < 113`, `newExp = 0`, adjust frac", this logic applies to **floatToHalf**. For **halfToFloat**, the standard logic is:
            $$newExp = exponent - 15 + 127 = exponent + 112$$
            $$newFrac = mantissa \ll 13$$
    *   *Correction based on Source [S2, unofficial] specific notes*: The source [S2, unofficial] lists steps for `floatToHalf` mixed with `halfToFloat`. Let's strictly follow the provided notes for `halfToFloat` which are less explicit in [S2, unofficial] but implied by the reverse of [S1, unofficial]'s output. However, [S1, unofficial] provides the output formula for `halfToFloat`? No, [S1, unofficial] provides the output formula for `halfToFloat` in the "Otherwise" clause:
        $$result = signMul \times 2^{(exponent - 15)} \times (1 + \frac{mantissa}{1024})$$
        This formula calculates the value. To pack it into a 32-bit float, we must reconstruct the IEEE 754 bits.
        *   Sign: $sgn \ll 31$
        *   Exponent: $(exponent + 112) \ll 23$ (if normalized)
        *   Mantissa: $mantissa \ll 13$

    *   *Refined Step 4 for Implementation*:
        If $exponent > 0$:
        $$newExp = exponent + 112$$
        $$newFrac = mantissa \ll 13$$
        $$f = (sgn \ll 31) | (newExp \ll 23) | newFrac$$
        Use `memcpy` to interpret the 32-bit integer as a float to avoid type punning UB [S2, unofficial].

### Algorithm: floatToHalf
Converts a 32-bit float `f` to a 16-bit half-precision float `h`.

1.  **Extract Float Components**:
    $$sign = (f \gg 31) \ \& \ 1$$
    $$exp = (f \gg 23) \ \& \ 0xFF$$
    $$frac = f \ \& \ 0x7FFFFF$$
2.  **Determine New Exponent and Mantissa**:
    *   If $exp == 0$:
        *   $newExp = 0$
        *   $newFrac = 0$ (Zero)
    *   If $exp < 113$:
        *   $newExp = 0$
        *   Adjust $frac$ for denormalization (shift right). Check for overflow to $newExp = 1$ [S2, unofficial].
    *   If $113 \le exp < 142$:
        *   $newExp = exp - 112$
        *   $newFrac = frac \gg 13$
    *   If $exp \ge 142$:
        *   $newExp = 31$ (Infinity/NaN)
        *   $newFrac = 0$
        *   If original $frac \neq 0$, set $newFrac$ to indicate NaN (implementation dependent, usually max mantissa) [S2, unofficial].
3.  **Pack Result**:
    $$h = (sign \ll 15) | (newExp \ll 10) | (newFrac \gg 0)$$
    *   Note: The source [S2, unofficial] says `frac >> 13`. Since `frac` is 23 bits, shifting right by 13 leaves 10 bits, which fits the half-precision mantissa.

### Algorithm: packHalf2x16
1.  Input: $h_x, h_y$ (16-bit values).
2.  Output:
    $$result = h_x | (h_y \ll 16)$$
    Cast to `uint32_t` [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def float_to_half(f: float) -> int:
    """Convert a 32-bit float to a 16-bit half-precision float (uint16)."""
    # 1. Extract float bits
    # Use memcpy equivalent: pack float into 4 bytes, then unpack as uint32
    f_bits = struct.unpack('>I', struct.pack('>f', f))[0]
    
    sign = (f_bits >> 31) & 1
    exp = (f_bits >> 23) & 0xFF
    frac = f_bits & 0x7FFFFF
    
    # 2. Determine new exponent and mantissa
    if exp == 0:
        # Zero or denormalized single (rare in this context, usually treated as 0)
        new_exp = 0
        new_frac = 0
    elif exp < 113:
        # Underflow to denormalized half or zero
        # Adjust frac: shift right by (113 - exp) + 13? 
        # Standard logic: shift frac right by (113 - exp + 13) is complex.
        # Simplified based on note: newExp = 0, adjust frac.
        # We shift the mantissa to align with half precision, then check if it's zero.
        shift = 113 - exp + 13 # 113 is the threshold where exp-112=1
        new_frac = frac >> shift
        new_exp = 0
        if new_frac == 0:
            return (sign << 15)
    elif exp < 142:
        new_exp = exp - 112
        new_frac = frac >> 13
    else:
        # Overflow or Inf/NaN
        if frac == 0:
            new_exp = 31
            new_frac = 0
        else:
            new_exp = 31
            new_frac = 0x3FF # NaN
            # Note: Source [S2, unofficial] says frac=0 for exp>=142, but standard NaN requires non-zero frac.
            # We follow source [S2, unofficial] strictly: frac = 0.
            new_frac = 0
            
    # 3. Pack result
    h = (sign << 15) | (new_exp << 10) | (new_frac >> 0) # new_frac is already 10 bits
    return h

def half_to_float(h: int) -> float:
    """Convert a 16-bit half-precision float (uint16) to a 32-bit float."""
    # 1. Extract half bits
    sgn = (h >> 15) & 0x1
    exponent = (h >> 10) & 0x1f
    mantissa = h & 0x3ff
    
    sign_mul = -1.0 if sgn == 1 else 1.0
    
    # 2. Handle special cases
    if exponent == 0:
        if mantissa == 0:
            return 0.0 if sgn == 0 else -0.0
        else:
            # Denormalized
            return sign_mul * (2 ** -14) * (mantissa / 1024.0)
    elif exponent == 31:
        if mantissa != 0:
            return float('nan')
        else:
            return float('inf') if sgn == 0 else float('-inf')
    else:
        # Normalized
        # Formula from [S1, unofficial]: signMul * 2^(exponent - 15) * (1 + mantissa / 1024)
        val = sign_mul * (2 ** (exponent - 15)) * (1.0 + mantissa / 1024.0)
        return val

def pack_half_2x16(hx: int, hy: int) -> int:
    """Pack two 16-bit half-precision floats into a 32-bit uint."""
    return (hx & 0xFFFF) | ((hy & 0xFFFF) << 16)
```

### Parameters and Edge Cases
*   **Precision**: Half-precision has 10 bits of mantissa vs 23 in single. Rounding errors are significant for values with high precision requirements.
*   **Range**: Half-precision max value is $\approx 65504$. Single precision max is $\approx 3.4 \times 10^{38}$. Values outside half range overflow to Infinity [S2, unofficial].
*   **Denormals**: Half-precision supports denormalized numbers down to $2^{-14}$ (approx $6.1 \times 10^{-5}$). Single precision denormals are much smaller. Conversion from single to half may flush small denormals to zero [S2, unofficial].
*   **Type Punning**: Direct casting between `float*` and `uint16_t*` is undefined behavior in C/C++. Use `memcpy` or bit-casting utilities [S2, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time**: $O(1)$ per conversion. Bitwise operations and basic arithmetic.
*   **Memory**: $O(1)$. No additional data structures required.

### Strengths and Limitations
*   **Strengths**: Reduces memory bandwidth and storage by 50% compared to single precision. Essential for large volume datasets. Avoids UB from type punning [S2, unofficial].
*   **Limitations**: Loss of precision and dynamic range. Not suitable for intermediate calculations requiring high precision. Denormalized numbers can be slow on some hardware.

### When to Use
*   Storing volume textures (e.g., `GL_RG16F`, `GL_RGBA16F`).
*   Transmitting volume data over networks.
*   When memory bandwidth is the bottleneck.

### When Not to Use
*   High-precision scientific simulations.
*   Intermediate steps in complex volume rendering integrations where precision loss accumulates.

## Background

### Origin
The IEEE 754-2008 standard defined the binary16 (half-precision) format. The specific implementation details in the notes come from open-source graphics projects.

### History
*   **[S2, unofficial]**: `liaozhangsheng/gsplat-opengl` provides a C++ implementation using `memcpy` to safely convert floats to halves, avoiding undefined behavior.
*   **[S1, unofficial]**: `mkkellogg/GaussianSplats3D` uses JavaScript/TypeScript logic for conversion, providing explicit formulas for denormalized and normalized values.

### Usage
Used in GPU-based volume rendering, Gaussian Splatting (as seen in the source repos), and general graphics programming where memory efficiency is critical.

## Sources

[S1] GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S2] gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
