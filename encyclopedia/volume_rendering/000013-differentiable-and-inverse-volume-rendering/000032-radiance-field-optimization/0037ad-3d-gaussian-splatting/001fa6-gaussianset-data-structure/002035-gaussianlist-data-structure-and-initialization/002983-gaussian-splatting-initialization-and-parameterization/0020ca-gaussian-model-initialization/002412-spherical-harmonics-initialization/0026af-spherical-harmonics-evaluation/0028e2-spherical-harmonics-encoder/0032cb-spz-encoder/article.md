# SPZ Encoder

Also called: SPZ Spherical Harmonics Encoder, SPZ SH Encoder

## Overview
The SPZ Encoder is a data serialization module that converts 3D Gaussian Splatting attributes, specifically Spherical Harmonics (SH) coefficients, into a compact binary format (SPZ). It solves the problem of efficiently storing and transmitting view-dependent color information by quantizing floating-point SH coefficients into fixed-point integers. It differs from the [Spherical Harmonics Encoder](4390-spherical-harmonics-encoder.md) by focusing on the final byte-array serialization and quantization rather than the mathematical basis expansion, and from [SH Feature Encoding](6642-sh-feature-encoding.md) by using a specific fixed-point scheme tailored for the SPZ format.

## How it works
**Inputs:**
- Spherical Harmonics coefficients (floating-point values).
- Maximum SH degree (integer).
- `fractional_bits`: Number of fractional bits for quantization (u8) [S1, unofficial].
- `version`: Optional version number for the SPZ format (u32) [S1, unofficial].

**Outputs:**
- A byte array containing the encoded SPZ data.

**Data Structures:**
The encoder manages internal state including:
- `maxShDegree`: The active SH degree, clamped to the specified maximum.
- `fractional_bits`: The precision parameter for quantization.
- `version`: The format version identifier.

**Steps:**
1. **Clamp Degree:** Clamp the input spherical harmonics degree to the specified maximum SH degree. Update the internal `maxShDegree` field to match this clamped value [S1, unofficial].
2. **Initialize:** Set up the encoder with the clamped `maxShDegree`, the `fractional_bits` parameter, and the optional `version` [S1, unofficial].
3. **Encode:** Execute the encoding process. This involves mapping the SH degree to the correct dimension count and quantizing the coefficients.
4. **Serialize:** Produce the final byte array [S1, unofficial].

**Dimension Mapping:**
The number of dimensions for SH coefficients is determined by the degree [S2, unofficial]:
- Degree 0: 0 dimensions
- Degree 1: 3 dimensions
- Degree 2: 8 dimensions
- Degree 3: 15 dimensions

**Quantization Formula:**
Floating-point SH coefficients are quantized to integers. The unquantized value $x$ is derived from the quantized integer $x_{int}$ as [S2, unofficial]:
$$ x = \frac{x_{int} - 128.0}{128.0} $$
Conversely, to encode a float $x$ to an integer $x_{int}$, the operation is:
$$ x_{int} = \text{round}(x \times 128.0 + 128.0) $$
The `fractional_bits` parameter controls the precision of this fixed-point representation [S1, unofficial].

**Pitfalls:**
- Unsupported SH degrees (e.g., > 3) log an error and return 0 dimensions [S2, unofficial].
- Precision loss occurs during quantization; the choice of `fractional_bits` affects visual fidelity vs. file size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_spz(sh_coefficients: List[float], max_sh_degree: int, fractional_bits: int, version: int = None) -> bytes:
    # Clamp SH degree
    clamped_degree = min(max_sh_degree, 3) # Max supported degree in SPZ is typically 3
    
    # Map degree to dimensions
    dim_map = {0: 0, 1: 3, 2: 8, 3: 15}
    num_dims = dim_map.get(clamped_degree, 0)
    
    if num_dims == 0:
        raise ValueError(f"Unsupported SH degree: {max_sh_degree}")
        
    # Initialize encoder state
    encoder = SPZEncoder(max_sh_degree=clamped_degree, fractional_bits=fractional_bits, version=version)
    
    # Quantize coefficients
    quantized_coeffs = []
    for coeff in sh_coefficients:
        # Quantize float to int: x_int = round(x * 128.0 + 128.0)
        # Note: The exact scaling factor may depend on fractional_bits implementation details [S1, S2, unofficial]
        # Standard SPZ uses 128.0 scale for 8-bit signed/unsigned range mapping
        x_int = round(coeff * 128.0 + 128.0)
        # Clamp to valid byte range (0-255 for unsigned, or -128 to 127 for signed depending on format)
        # SPZ typically uses unsigned bytes
        x_int = max(0, min(255, x_int))
        quantized_coeffs.append(x_int)
        
    # Serialize to byte array
    # The exact layout (e.g., header, degree byte, then coefficients) is implementation-specific [S1, unofficial]
    # Assuming a simple concatenation for this example
    byte_array = bytearray()
    byte_array.append(clamped_degree)
    for q in quantized_coeffs:
        byte_array.append(q)
        
    return bytes(byte_array)
```

## Subtopics

### SPZ Spherical Harmonics Degree Dimension Mapping
Maps SH degree to dimension count: Degree 0 -> 0 dimensions, Degree 1 -> 3 dimensions, Degree 2 -> 8 dimensions, Degree 3 -> 15 dimensions. Unsupported SH degrees log an error and return 0 [S2, unofficial].

### SPZ Spherical Harmonics Unquantization
Reconstructs floating-point coefficients from quantized integers using the formula $x = \frac{x_{int} - 128.0}{128.0}$ [S2, unofficial].

## Operating characteristics
**Time Complexity:** Linear with respect to the number of SH coefficients $O(N)$, where $N$ is the total number of coefficients to encode.
**Memory Complexity:** Linear with respect to the number of coefficients, storing one byte per coefficient (plus a small constant header).
**Strengths:** Compact storage for view-dependent colors; fast encoding/decoding due to simple arithmetic operations.
**Limitations:** Quantization introduces error; limited to SH degrees 0-3.
**When to use:** When storing or transmitting 3D Gaussian Splatting data in the SPZ format.
**When not to use:** When higher precision is required or when SH degrees > 3 are needed.

## Background
The SPZ Encoder is part of the SPZ (Splat Pack) format, an unofficial binary format for 3D Gaussian Splatting data. It is implemented in libraries such as `sparkjsdev/spark` and `mkkellogg/GaussianSplats3D` to facilitate efficient loading and rendering of Gaussian splats in web-based viewers [S1, S2, unofficial].

## Sources
[S1] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
[S2] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
