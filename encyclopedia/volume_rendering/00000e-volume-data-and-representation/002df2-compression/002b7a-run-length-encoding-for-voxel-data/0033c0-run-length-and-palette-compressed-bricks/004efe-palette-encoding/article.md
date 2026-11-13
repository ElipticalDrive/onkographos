# Palette Encoding

Also called: Pattern Encoding, PatternFixupID

## Overview
Palette Encoding is a sub-method of run-length and palette-compressed bricks that retrieves a palette ID (0 or 1) for a texel index based on a pattern number [S1, unofficial]. It serves as the final decoding step in BC6H P2 compression, distinguishing between two primary color values within a block [S1, unofficial]. It is distinct from the initial pattern selection logic, focusing solely on the binary classification of texels.

## How it works
**Inputs:**
- `p`: The pattern index (integer).
- `i`: The texel index within the block (0–15 for 4x4 blocks).

**Outputs:**
- `palette_id`: The palette ID (0 or 1) for the texel.

**Data Structures:**
- `p2`: Intermediate value, `p // 2`.
- `p3`: Intermediate value, `p % 2`.

**Steps:**
1. Decompose the pattern index `p` into `p2 = p / 2` and `p3 = p % 2` using integer division and modulo [S1, unofficial].
2. Select an encoded value based on `p2` using a switch-like structure with bitmasks [S1, unofficial]. This step determines the base pattern configuration.
3. If `p3` is 1, shift the selected encoded value right by 16 bits [S1, unofficial].
4. Extract the bit at position `i` from the resulting value [S1, unofficial].
5. Return the extracted bit as the `palette_id` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_palette_id(p: int, i: int) -> int:
    # Step 1: Decompose pattern index
    p2 = p // 2
    p3 = p % 2
    
    # Step 2: Select encoded value based on p2
    # Note: The specific bitmask logic for p2 is not detailed in the source.
    # This is a placeholder for the switch-like structure.
    encoded_value = select_encoded_value(p2) 
    
    # Step 3: Shift if p3 is 1
    if p3 == 1:
        encoded_value = encoded_value >> 16
        
    # Step 4: Extract bit i
    palette_id = (encoded_value >> i) & 1
    
    # Step 5: Return result
    return palette_id

def select_encoded_value(p2: int) -> int:
    # Placeholder for the switch-like structure using bitmasks
    # The source does not provide the specific mapping or bitmasks.
    # This function must be implemented based on the BC6H specification details not present in the provided notes.
    return 0
```

**Edge Cases and Pitfalls:**
- The source does not specify the exact bitmasks or switch-case logic for `p2`. Implementers must refer to the full BC6H specification or the original GPURealTimeBC6H source code for the complete `select_encoded_value` logic [S1, unofficial].
- Precision issues are unlikely as the operations are integer-based.
- Boundary conditions for `i` (0–15) must be respected.

## Subtopics

## Operating characteristics
**Time Complexity:** O(1) per texel.
**Memory Complexity:** O(1).
**Strengths:** Simple, fast retrieval of palette IDs.
**Limitations:** Requires the full pattern index and bit extraction logic.

## Background
Originating from the BC6H texture compression standard, Palette Encoding is used in GPU-based real-time compression implementations [S1, unofficial]. It is part of the P2 (Pattern 2) encoding mode, which uses two primary colors and a palette to represent block data [S1, unofficial].

## Sources
[S1] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
