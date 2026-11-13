# Palette Encoding

Also called: Pattern Encoding, PatternFixupID

## Overview
Palette Encoding is a sub-method of run-length and palette-compressed bricks used to decode texel values within a compressed block. It retrieves a palette ID (0 or 1) for a specific texel index based on a predefined binary pattern. It sets itself apart from sibling methods by using a deterministic pattern lookup to select between two encoded values, rather than interpolating between endpoints or using fixed geometric splits.

## How it works
**Inputs:** A pattern index $p$ (integer), and the encoded data structure containing two values (Value 0 and Value 1).
**Outputs:** The selected encoded value for the texel.

**Data Structures:**
The encoded block contains two primary values, $V_0$ and $V_1$. In the context of BC6H P2, these are 3D line segments in RGB space, but the Palette Encoding step specifically selects which segment's parameters apply to a texel.

**Steps:**
1. Decompose the pattern index $p$ into two components:
   $$ p_2 = \lfloor p / 2 \rfloor $$
   $$ p_3 = p \pmod 2 $$
2. Select the base encoded value based on $p_2$. This acts as a switch using bitmasks to determine which of the two primary values ($V_0$ or $V_1$) is the candidate.
3. If $p_3$ is 1, shift the selected value right by 16 bits. This operation extracts the upper 16 bits, effectively applying a "fixup" or selecting the second part of a split value.
4. Extract bit $i$ from the result if further refinement is required by the parent block structure.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def palette_encode_decode(p, value_0, value_1):
    """
    Decode a texel value using Palette Encoding.
    
    Args:
        p (int): Pattern index.
        value_0 (int): First encoded value (32-bit or larger).
        value_1 (int): Second encoded value (32-bit or larger).
        
    Returns:
        int: The selected and potentially shifted encoded value.
    """
    # Decompose pattern index
    p2 = p // 2
    p3 = p % 2
    
    # Select encoded value based on p2
    # Using a switch-like structure with bitmasks
    if p2 == 0:
        selected = value_0
    else:
        selected = value_1
        
    # Shift right by 16 if p3 is 1
    if p3 == 1:
        selected = selected >> 16
        
    # Extract bit i (context-dependent, often part of larger reconstruction)
    # For the purpose of this step, we return the selected value
    return selected
```

**Parameters:**
- $p$: Pattern index. Typical values depend on the block size and pattern set. Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision: Shifting right by 16 bits discards lower bits. Ensure the source values have sufficient precision in the upper bits if the lower bits are significant.
- Boundaries: Ensure $p$ is within the valid range for the pattern set.

## Subtopics

### [BC6H P2 Pattern Encoding](14204-bc6h-p2-pattern-encoding.md)
BC6H P2 Pattern Encoding compresses a 4x4 texel block by partitioning the 16 texels into two subsets (partitions 0 and 1) based on a predefined binary pattern, then encoding each subset with a separate 3D line segment in RGB space. It solves the problem of encoding blocks with bimodal color distributions where a single linear interpolation (P1) is insufficient. It sets itself apart from P1 encoding by using two endpoints per color channel per partition, allowing for more complex color transitions, and from P0 encoding by allowing arbitrary partitioning patterns rather than fixed geometric splits.

## Operating characteristics
- Time complexity: $O(1)$ per texel, as it involves simple arithmetic and bitwise operations.
- Memory complexity: $O(1)$ additional memory per texel.
- Strengths: Simple to implement, low computational overhead.
- Limitations: Limited to two values per block, which may not capture complex color distributions.
- When to use: When the data can be approximated by two distinct values or segments.
- When not to use: When the data has more than two distinct modes or requires finer interpolation.

## Background
Origin: The method is described in the context of BC6H compression, specifically in the GPURealTimeBC6H repository.
History: BC6H is a standard for compressing HDR floating-point textures. The Palette Encoding method is part of the P2 partitioning scheme.
Usage: Used in real-time graphics applications for compressing HDR textures.

## Sources
[S1] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
