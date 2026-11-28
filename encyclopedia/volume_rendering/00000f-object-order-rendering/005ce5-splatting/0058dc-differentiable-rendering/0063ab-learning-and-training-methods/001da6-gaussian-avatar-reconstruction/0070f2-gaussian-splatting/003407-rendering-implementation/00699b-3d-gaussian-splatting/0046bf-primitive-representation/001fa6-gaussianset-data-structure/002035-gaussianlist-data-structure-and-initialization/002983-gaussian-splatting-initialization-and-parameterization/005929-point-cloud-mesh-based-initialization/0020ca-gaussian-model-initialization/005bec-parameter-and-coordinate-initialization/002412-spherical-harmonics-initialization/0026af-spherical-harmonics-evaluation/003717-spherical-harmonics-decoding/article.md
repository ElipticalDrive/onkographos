# Spherical Harmonics Decoding

Also called: RGB SH decoding, SH coefficient unpacking

## Overview
Spherical Harmonics (SH) Decoding is the process of extracting and converting stored SH coefficients into view-dependent RGB color values for 3D Gaussian splats. It solves the problem of interpreting compressed, packed coefficient data into a format suitable for rendering. It differs from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md) by focusing on the data layout and unpacking mechanics rather than the mathematical basis function evaluation, and from [Spherical Harmonics Rotation](3985-spherical-harmonics-rotation.md) by assuming the coefficients are already in the camera space or handling the unpacking prior to rotation.

## How it works
**Inputs:**
- A flat array of SH coefficients, typically stored as `Uint32` values.
- The maximum SH degree ($L$) used for the splat (typically 0, 1, 2, or 3).
- A mapping table (`SH_KEYS`) that maps coefficient indices to their corresponding SH basis index.

**Outputs:**
- An array of `THREE.Color` objects (or equivalent RGB structures), one for each SH coefficient, representing the unpacked color components.

**Data Structures:**
- **Packed Coefficient Array:** A flat `Uint32Array` where coefficients are packed 4 per 32-bit word.
- **SH Keys Map:** An array `SH_KEYS` where `SH_KEYS[i]` returns the SH basis index for the $i$-th coefficient in the sequence.
- **Result Array:** An array of color objects, initialized to length equal to the number of coefficients.

**Number of Coefficients:**
The total number of coefficients $N$ for a given degree $L$ is calculated as:
$$ N = L \times (L + 2) $$
For $L=0$, $N=0$ (or 1 if constant term is stored separately, but standard SH degree 0 implies 1 coefficient $Y_{0,0}$). The source notes imply a formula `degree * (degree + 2)`. For $L=1$, $N=3$. For $L=2$, $N=8$. For $L=3$, $N=15$. Note: Standard SH theory says $N = (L+1)^2$. The source formula `degree * (degree + 2)` yields $0, 3, 8, 15$ for $L=0,1,2,3$. This matches $(L+1)^2 - 1$ for $L \ge 1$, suggesting the constant term ($L=0$) might be handled separately or the formula in the source is specific to the implementation's indexing. We follow the source's explicit instruction: `degree * (degree + 2)`.

**Steps:**
1. **Initialize:** Create a result array of size $N$. Initialize an RGB accumulator to $[0, 0, 0]$.
2. **Iterate Coefficients:** For each coefficient index $c$ from $0$ to $N-1$:
   a. **Determine Source Index:** Calculate the index in the packed array using `SH_KEYS[c]`. Let this be `idx`.
   b. **Determine Word Offset:** Calculate the offset within the 32-bit word using `offset = c & 3`. This corresponds to the channel (R, G, B, or unused/padding) within the packed word.
   c. **Read Word:** Access the packed array at `idx`. If the index is out of bounds or the value is missing, default to $0$.
   d. **Decode RGB:** Extract the RGB components from the 32-bit word. The source mentions `decodeShRgbToArray`, which implies a specific bit-packing scheme. Typically, SH coefficients are stored as 16-bit floats or fixed-point. In a `Uint32` array, 4 coefficients might be packed. The offset `c & 3` selects which 8-bit or 16-bit chunk corresponds to R, G, or B. The source does not specify the exact bit-width per channel, but implies a direct mapping.
   e. **Store Color:** Create a new `THREE.Color` object from the decoded RGB values and store it in the result array at index $c$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_sh_coefficients(packed_coeffs, sh_keys, degree):
    """
    Decodes packed SH coefficients into an array of Color objects.
    
    Args:
        packed_coeffs: Uint32Array of packed SH data.
        sh_keys: Array mapping coefficient index to packed array index.
        degree: Maximum SH degree (0, 1, 2, or 3).
        
    Returns:
        List of Color objects.
    """
    # Calculate number of coefficients based on source formula
    num_coeffs = degree * (degree + 2)
    
    # Handle degree 0 case if necessary (source formula gives 0)
    if degree == 0:
        num_coeffs = 1  # [G] Standard SH has 1 coefficient for L=0. Source formula gives 0, which is likely an error or specific to non-constant terms. We assume 1 for correctness in rendering.
        
    result = []
    
    for c in range(num_coeffs):
        # Determine source array index
        idx = sh_keys[c]
        
        # Determine word offset (channel within the packed word)
        offset = c & 3
        
        # Read the word, default to 0 if missing
        if idx < len(packed_coeffs):
            word = packed_coeffs[idx]
        else:
            word = 0
            
        # Decode RGB from the word
        # [G] The exact bit-packing is not specified in the source.
        # Assuming a standard 8-bit per channel or 16-bit float packing.
        # The source mentions `decodeShRgbToArray`, which is a helper.
        # We assume a generic unpacking for demonstration.
        r, g, b = unpack_rgb_from_word(word, offset)
        
        # Store as Color object
        color = Color(r, g, b)
        result.append(color)
        
    return result

def unpack_rgb_from_word(word, offset):
    """
    Unpacks RGB from a 32-bit word based on offset.
    [G] Placeholder for specific bit-packing logic.
    """
    # Example: 8-bit per channel, little-endian
    # offset 0: R, offset 1: G, offset 2: B, offset 3: Unused
    if offset == 0:
        r = word & 0xFF
        g = (word >> 8) & 0xFF
        b = (word >> 16) & 0xFF
    elif offset == 1:
        r = (word >> 8) & 0xFF
        g = (word >> 16) & 0xFF
        b = (word >> 24) & 0xFF
    elif offset == 2:
        r = (word >> 16) & 0xFF
        g = (word >> 24) & 0xFF
        b = 0 # [G] Assuming unused or zero
    else:
        r, g, b = 0, 0, 0
    return r, g, b
```

**Parameters:**
- `degree`: Integer, typically 0, 1, 2, or 3.
- `sh_keys`: Array of integers, mapping coefficient index to packed array index.

**Edge Cases:**
- **Missing Coefficients:** If `idx` is out of bounds, default to 0.
- **Degree 0:** The source formula `degree * (degree + 2)` yields 0 for $L=0$. This is likely a bug or specific to the implementation's handling of the constant term. Standard SH has 1 coefficient for $L=0$. [G] We assume 1 coefficient for $L=0$ for correctness.
- **Precision:** The source uses `Uint32`, implying fixed-point or packed float representation. Precision loss may occur if not handled correctly.

## Subtopics

### [Spherical Harmonics Degree 0 Color Reconstruction](11528-spherical-harmonics-degree-0-color-reconstruction.md)
Reconstructs base diffuse color from $l=0$ coefficient for fallback rendering. It differs by evaluating only the constant basis function, providing direction-independent color. Choose it for glTF viewers or point-cloud visualization.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of coefficients. Linear in the number of coefficients.
- **Memory Complexity:** $O(N)$ for the result array.
- **Performance:** Fast, as it involves simple array access and bit manipulation.
- **Strengths:** Efficient unpacking of packed data.
- **Limitations:** Depends on the specific packing scheme. Not given in the sources: exact bit-packing details.

## Background
Originated in the context of 3D Gaussian Splatting implementations, specifically in tools like `Gaussian-Splat-Lite` [S1, unofficial]. It is used to decode SH coefficients stored in a compact format for efficient rendering.

## Sources
[S1] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
