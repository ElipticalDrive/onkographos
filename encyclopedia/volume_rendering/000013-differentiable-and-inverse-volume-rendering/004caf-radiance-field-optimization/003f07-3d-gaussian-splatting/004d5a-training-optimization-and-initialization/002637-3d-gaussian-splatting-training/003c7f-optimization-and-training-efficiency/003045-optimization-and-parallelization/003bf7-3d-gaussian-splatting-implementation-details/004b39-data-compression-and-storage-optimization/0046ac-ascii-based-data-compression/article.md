# ASCII-Based Data Compression
Also called: Anywhere Compression, Base 94 Floating-Point Compression, Base 95 Integer Compression

## Overview
This method accelerates data transmission during 3D Gaussian Splatting training by compressing numerical attributes into ASCII strings using base-95 and base-94 encoding schemes [S1, unofficial]. It distinguishes itself from sibling methods like Sensitivity-Aware Vector Clustering by focusing on transmission bandwidth reduction through character-level encoding rather than spatial redundancy or vector quantization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Numerical values (integers or floating-point numbers) representing 3D Gaussian attributes (position, scale, rotation, SH coefficients) and color matrix values.
*   **Output:** Compressed ASCII strings for transmission; decoded numerical values for rendering/training.

**Data Structures:**
*   **Integer Encoding:** A mapping table of 95 characters (ASCII 32–126) representing values in the range $[0, 95]$.
*   **Floating-Point Encoding:** A mapping table of 94 characters (ASCII 32–125) for the integer component, plus a delimiter character (ASCII 126, `~`) to separate integer and fractional parts.

**Steps:**
1.  **Integer Compression:**
    *   For an integer value $v$ in the range $[0, 95]$, map $v$ to the corresponding ASCII character $c$ where $c = \text{chr}(v + 32)$.
    *   Store $c$ as the compressed representation.
    *   *Decoding:* Convert the ASCII character $c$ back to integer $v = \text{ord}(c) - 32$.

2.  **Floating-Point Compression:**
    *   Decompose the floating-point number $f$ into an integer part $I$ and a decimal place count $d$.
    *   *Note on Formula:* The source defines the reconstruction as $f = I \times 10^{-d}$ [S1, unofficial]. This implies $d$ represents the number of decimal places, not the fractional value itself.
    *   Map the integer part $I$ (assuming range $[0, 94]$) to an ASCII character $c_I = \text{chr}(I + 32)$.
    *   Map the decimal place count $d$ to an ASCII character $c_d = \text{chr}(d + 32)$ (assuming $d$ fits in the base-94 range).
    *   Concatenate $c_I$, the delimiter character $c_{del} = \text{chr}(126)$, and $c_d$ to form the string.
    *   *Decoding:* Split the string at the delimiter. Convert $c_I$ to $I$ and $c_d$ to $d$. Calculate $f = I \times 10^{-d}$.

3.  **Color Matrix Optimization:**
    *   Quantize each item in the output color matrix to one decimal place within the range $[0.0, 2.0]$.
    *   This results in 21 discrete alternative numbers ($0.0, 0.1, \dots, 2.0$).
    *   Apply ASCII compression to these quantized values to reduce data size significantly [S1, unofficial].

**Parameters:**
*   **Integer Base:** 95 (ASCII 32–126).
*   **Floating-Point Integer Base:** 94 (ASCII 32–125).
*   **Delimiter:** ASCII 126 (`~`).
*   **Color Precision:** 1 decimal place.

**Edge Cases and Pitfalls:**
*   **Matrix Traversal:** Compressing large color matrices via ASCII string conversion is computationally expensive due to traversal overhead [S1, unofficial].
*   **Delimiter Usage:** Using line break characters as delimiters adds extra operators in writing and decoding, increasing complexity [S1, unofficial].
*   **Performance Limit:** The method yields approximately 6% or less improvement in training speed [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_integer(value: int) -> str:
    """
    Compress an integer in range [0, 95] to a single ASCII character.
    """
    if not (0 <= value <= 95):
        raise ValueError("Integer out of range [0, 95]")
    # ASCII 32 corresponds to value 0
    return chr(value + 32)

def decompress_integer(char: str) -> int:
    """
    Decompress a single ASCII character to an integer.
    """
    return ord(char) - 32

def compress_float(value: float) -> str:
    """
    Compress a floating-point number.
    Assumes value can be represented as I * 10^-d where I is in [0, 94]
    and d is the number of decimal places.
    """
    # Determine integer part I and decimal places d
    # Note: The source formula is f = I * 10^-d.
    # This implies we need to extract I and d such that this holds.
    # For example, if value is 12.34, this encoding scheme as described
    # is ambiguous without a specific rounding/truncation rule for I and d.
    # Based on "one decimal place" context for color, we assume d is fixed
    # or derived from precision. However, the general float compression
    # selects chars for I and d separately.
    
    # Simplified interpretation for implementation:
    # 1. Round/Truncate to integer I.
    # 2. Determine d (number of decimal places needed to reconstruct).
    # The source does not specify how d is calculated for arbitrary floats,
    # only that it is encoded. We assume d is a small integer representing
    # the scale factor exponent.
    
    # Let's assume a standard fixed-point interpretation for the example:
    # If value = 12.3, I=12, d=1. f = 12 * 10^-1 = 1.2 (Lossy/Specific format).
    # The source says "decimal places d".
    
    # To implement strictly from source:
    # We need I and d.
    # Let's assume the input is pre-processed or d is known/fixed for the attribute.
    # For general implementation, we might fix d=1 for color, but for general floats?
    # The source is vague on arbitrary float handling. We implement the encoding
    # of I and d as described.
    
    # Placeholder for extracting I and d from value based on specific application logic
    # [G] Assuming value is scaled such that I is the integer part and d is the
    # number of decimal digits in the fractional part, or a fixed precision.
    # For color matrix (1 decimal place):
    # value = 1.2 -> I=1, d=1? No, 1 * 10^-1 = 0.1.
    # The formula f = I * 10^-d suggests I is the significant digits and d is the shift.
    # Example: 12.3 -> I=123, d=1? 123 * 0.1 = 12.3.
    # But I must be in [0, 94]. So this encoding is limited to small numbers or
    # requires I to be the integer part and d to be the decimal part count?
    # Source: "integer component ... range [0, 94] ... decimal places d".
    # This implies I is the integer part of the float.
    # If value = 5.5, I=5. What is d? "decimal places d".
    # If d=1, f = 5 * 10^-1 = 0.5. This does not reconstruct 5.5.
    # There is a contradiction or missing detail in the source regarding reconstruction
    # of the fractional part.
    # However, for Color Matrix: "one decimal place ... range [0.0, 2.0]".
    # Values are 0.0 to 2.0.
    # If value=1.5, I=1. d=1? f=0.1. Incorrect.
    # Perhaps d is not the exponent, but the fractional part encoded?
    # "split ... integer part I ... decimal places d".
    # "f = I * 10^-d".
    # This formula is likely a typo in the source or refers to a specific scientific notation
    # where I is the mantissa. But I is limited to 94.
    # Given the ambiguity, we implement the encoding of I and d as characters.
    
    # [G] Implementation note: The source's reconstruction formula f = I * 10^-d
    # is inconsistent with standard decimal representation if I is just the integer part.
    # It likely implies I is the significant integer and d is the decimal shift.
    # For the purpose of this article, we encode I and d as characters.
    
    # Extract I (integer part)
    I = int(value)
    if not (0 <= I <= 94):
        raise ValueError("Integer part out of range [0, 94]")
    
    # Extract d (number of decimal places)
    # [G] The source does not specify how d is derived for general floats.
    # For color matrix (1 decimal place), d=1.
    # We assume d is provided or fixed by the attribute type.
    # Here we assume d is the number of decimal digits in the fractional part.
    str_val = f"{value:.1f}" # Assume 1 decimal place for simplicity in this context
    if '.' in str_val:
        d = len(str_val.split('.')[1])
    else:
        d = 0
        
    if not (0 <= d <= 94): # Assuming d also fits in base 94
        raise ValueError("Decimal places out of range")
        
    char_I = chr(I + 32)
    char_d = chr(d + 32)
    delimiter = chr(126)
    
    return f"{char_I}{delimiter}{char_d}"

def decompress_float(s: str) -> float:
    """
    Decompress an ASCII string to a floating-point number.
    """
    delimiter = chr(126)
    if delimiter not in s:
        raise ValueError("Invalid format")
    
    parts = s.split(delimiter)
    if len(parts) != 2:
        raise ValueError("Invalid format")
        
    char_I = parts[0]
    char_d = parts[1]
    
    I = ord(char_I) - 32
    d = ord(char_d) - 32
    
    # Reconstruction formula from source
    f = I * (10 ** -d)
    return f
```

## Operating characteristics
*   **Time Complexity:** Encoding/Decoding is $O(1)$ per number, but matrix traversal for color compression is $O(N)$ where $N$ is the number of elements, and is identified as a bottleneck [S1, unofficial].
*   **Memory Complexity:** Reduces storage size by replacing multi-byte floats/ints with single ASCII characters (1 byte) where applicable [S1, unofficial].
*   **Performance:** Gains approximately 6% or less in training speed [S1, unofficial].
*   **Strengths:** Simple implementation; reduces data transmission size.
*   **Limitations:** Limited range for integer/floating-point components (0-95); color matrix compression is slow due to traversal overhead; limited overall speedup [S1, unofficial].

## Background
This method was proposed in the context of improving 3D Gaussian Splatting training speed by optimizing data transmission [S1, unofficial]. It is part of the "Anywhere" compression approach described in the I3DGS paper [S1, unofficial].

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
