# ASCII-Based Data Compression
Also called: Anywhere Compression, Base 94 Floating-Point Compression, Base 95 Integer Compression

## Overview
This method accelerates data transmission during the training of 3D Gaussian Splatting models by compressing floating-point and integer values into ASCII strings [S1, unofficial]. It differs from sibling methods like [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md) and [Compact 3D Gaussian Representation (C3DGS)](12927-compact-3d-gaussian-representation-c3dgs.md) by operating on the serialization layer rather than the geometric representation or clustering logic, specifically targeting the bottleneck of transferring model parameters between devices or processes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Floating-point numbers (e.g., Gaussian parameters) and integer numbers (e.g., indices or counts) intended for data transmission.
*   **Output:** ASCII-encoded strings representing the compressed values.

**Data Structures:**
*   **Integer Encoding:** Maps integers in the range $[0, 95]$ to a single ASCII character.
*   **Floating-Point Encoding:** Maps the integer component to a single ASCII character in the range $[0, 94]$ and uses a delimiter to separate the fractional part.

**Steps:**

1.  **Integer Compression (Base 95):**
    *   Select ASCII characters with codes 32 to 126.
    *   Map each integer value $v \in [0, 95]$ to the character at index $v$ within this range.
    *   Result: A single character representing the integer.

2.  **Floating-Point Compression (Base 94):**
    *   Select ASCII characters with codes 32 to 125 for the integer component.
    *   Select ASCII character 126 (`~`) as the delimiter connecting the integer and fractional parts.
    *   Encode the integer part $I$ of the floating-point number using the Base 94 mapping.
    *   Encode the decimal places $d$ (number of decimal digits) as a separate integer or string component. *Note: The source describes splitting the string into integer part $I$ and decimal places $d$, then calculating $f = I \times 10^{-d}$.*

3.  **Decoding:**
    *   **Integer:** Convert the ASCII character directly back to the decimal integer.
    *   **Floating-Point:** Split the ASCII string at the delimiter (ASCII 126). Let the left part be the integer component $I$ and the right part represent the decimal precision $d$. Calculate the final value $f$ using the formula:
        $$ f = I \times 10^{-d} $$

4.  **Color Matrix Optimization:**
    *   The color matrix values are constrained to the range $[0.0, 2.0]$ with only one decimal place.
    *   This reduces the possible values to 21 alternatives ($0.0, 0.1, \dots, 2.0$).
    *   These values are compressed using the ASCII compression algorithm to significantly reduce data size.

**Parameters:**
*   **Integer Base:** 95 (ASCII 32–126).
*   **Floating-Point Integer Base:** 94 (ASCII 32–125).
*   **Delimiter:** ASCII 126 (`~`).
*   **Color Precision:** 1 decimal place.

**Pitfalls and Edge Cases:**
*   **Matrix Traversal Overhead:** Compressing large color matrices via ASCII string conversion is computationally expensive due to the traversal time required. Using PyTorch internal functions like `torch::where` and `torch::index_select` to replace explicit loops improves speed but is not fast enough to fully replace standard transmission methods [S1, unofficial].
*   **Line Breaks:** Using line break characters in ASCII strings adds extra operators during writing and decoding, increasing overhead [S1, unofficial].
*   **Densification:** Adjusting the `densify_and_prune` process, specifically removing `densify_and_split` operations where appropriate, can improve speed without impacting rendering quality [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_integer_base95(value: int) -> str:
    """
    Compress an integer in range [0, 95] to a single ASCII character.
    ASCII 32 maps to 0, ASCII 126 maps to 95.
    """
    if not (0 <= value <= 95):
        raise ValueError("Integer out of range [0, 95]")
    # ASCII 32 is the start of the range
    return chr(value + 32)

def decompress_integer_base95(char: str) -> int:
    """
    Decompress a single ASCII character back to an integer.
    """
    code = ord(char)
    if not (32 <= code <= 126):
        raise ValueError("Invalid ASCII character for integer compression")
    return code - 32

def compress_float_base94(value: float) -> str:
    """
    Compress a floating-point number.
    Integer part is encoded in Base 94 (ASCII 32-125).
    Delimiter is ASCII 126 (~).
    Fractional part is handled by specifying decimal places d.
    Formula: f = I * 10^-d
    """
    # Determine decimal places d to preserve precision
    # Note: The source implies d is the number of decimal places.
    # For simplicity in this implementation, we assume a fixed precision or
    # derive d from the input string representation if needed.
    # Here we assume the caller provides the precision or we round to 1 decimal.
    
    # Extract integer part I
    I = int(value)
    
    # Determine d (decimal places). 
    # If value is 1.5, I=1, d=1. f = 1 * 10^-1 = 0.1? 
    # Wait, the source says f = I * 10^-d. 
    # If I=1 and d=1, f=0.1. This implies I is the significant digits?
    # Or is I the integer part and d is the scale?
    # Source: "integer part I ... decimal places d ... f = I * 10^-d"
    # Example: 1.5 -> I=1, d=1? 1 * 0.1 = 0.1 != 1.5.
    # Re-reading: "split into two part ... integer part I ... decimal places d"
    # It is likely I represents the significant digits or the integer part 
    # and d represents the position. 
    # However, strictly following the formula f = I * 10^-d:
    # If value is 1.5, and we want to store it, maybe I=15, d=1?
    # But the source says "integer component ... range [0, 94]".
    # This suggests I is small. 
    # Let's stick to the literal formula provided: f = I * 10^-d.
    # This formula seems to describe a specific encoding where I is the 
    # integer part and d is the number of decimal digits, but the math 
    # f = I * 10^-d only works if I is the truncated integer and we are 
    # approximating, OR if I is the significant digits.
    # Given the constraint "integer component ... range [0, 94]", I is small.
    # Let's assume the standard interpretation: I is the integer part, 
    # and d is the number of decimal places, but the formula in the source 
    # might be a simplified representation of "I followed by d decimal digits".
    # However, to implement "from text alone", we must follow the formula.
    # If the formula is strictly f = I * 10^-d, then for 1.5, I=1, d=-1? No.
    # Let's assume the source meant: The string is "I" + "~" + "d_digits".
    # And the value is reconstructed by parsing I and the digits.
    # But the source explicitly gives the formula.
    # We will implement the formula as stated, noting the potential ambiguity.
    
    # To make I fit in [0, 94], the integer part must be small.
    # If the value is large, this compression is not applicable as described.
    
    # Let's assume d is the number of decimal places we want to store.
    # And I is the integer part.
    # The formula f = I * 10^-d is likely a typo in the source for 
    # "I concatenated with d decimal digits" or similar.
    # However, without clarification, we cannot guess.
    # We will implement the decoding as described: split by ~, get I and d.
    # And apply f = I * 10^-d.
    
    # For encoding, we need to determine I and d.
    # If value = 1.5, I = 1. d = 1? f = 0.1. Error.
    # If value = 1.5, I = 15, d = 1? f = 1.5. But I=15 is in [0,94].
    # So maybe I is the significant digits?
    # "integer component of the floating-point number" usually means the part before the dot.
    # This is a contradiction in the source notes.
    # We will implement the decoding logic as the primary source of truth for the format.
    
    # Encoding strategy based on decoding:
    # 1. Convert float to string with 1 decimal place (as per color matrix optimization).
    # 2. Split into integer part and decimal part.
    # 3. If the source implies I is the integer part, and d is the decimal digits count,
    #    the formula f = I * 10^-d is incorrect for reconstruction.
    #    We will assume the source meant: Value = I + (decimal_digits / 10^d).
    #    But we must follow the source.
    #    Let's assume the "integer component" I is actually the significant integer 
    #    formed by the digits? No, "integer component" is specific.
    
    # Given the ambiguity, we provide the decoding function which is explicit.
    # Encoding is marked as [G] for the specific mapping of float to I and d 
    # due to the formula mismatch.
    pass

def decompress_float_base94(s: str) -> float:
    """
    Decompress an ASCII string back to a float.
    Format: "IntegerChar~DecimalPlacesChar"
    Formula: f = I * 10^-d
    """
    if '~' not in s:
        raise ValueError("Missing delimiter")
    
    parts = s.split('~')
    if len(parts) != 2:
        raise ValueError("Invalid format")
    
    # Decode I
    I_char = parts[0]
    if len(I_char) != 1:
        raise ValueError("Integer part must be one character")
    I = decompress_integer_base95(I_char)
    
    # Decode d
    d_char = parts[1]
    # d is also an integer, likely encoded similarly or just the count?
    # Source says "ASCII string for the decimal places d".
    # If d is a count, it's an integer.
    # We assume d is encoded in Base 95 as well for consistency, 
    # or it's a single digit string.
    # Let's assume d is a single character encoded in Base 95.
    d = decompress_integer_base95(d_char)
    
    # Apply formula
    f = I * (10 ** -d)
    return f
```

## Operating characteristics
*   **Performance:** Gains approximately 6% or less improvement in training speed [S1, unofficial].
*   **Limitations:** Not suitable for compressing large color matrices directly via ASCII strings due to high traversal overhead, even with optimized PyTorch functions (`torch::where`, `torch::index_select`) [S1, unofficial].
*   **Use Case:** Best for transmitting smaller integer and floating-point parameters where the overhead of ASCII conversion is outweighed by reduced data size or transmission time.

## Background
This method was introduced in the context of improving 3D Gaussian Splatting training speed from multiple dimensions, including compression [S1, unofficial]. It is part of the "Anywhere" compression approach proposed to address data transmission bottlenecks.

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
