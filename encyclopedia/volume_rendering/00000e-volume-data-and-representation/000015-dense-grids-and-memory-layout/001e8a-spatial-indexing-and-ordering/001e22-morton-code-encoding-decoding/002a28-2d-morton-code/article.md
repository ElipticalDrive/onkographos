# 2D Morton Code

Also called: 2D Morton Index, 2D Z-order

## Overview
The 2D Morton Code maps two-dimensional grid coordinates $(x, y)$ into a one-dimensional integer index by interleaving their binary representations. This method preserves spatial locality, meaning that points close in 2D space typically have similar Morton indices, which optimizes cache performance and enables efficient spatial indexing for dense grids. It differs from 3D Morton codes by operating on only two dimensions and from Hilbert curves by using simple bit interleaving rather than recursive space-filling curve traversal, making it computationally cheaper but with slightly lower locality preservation.

## How it works
**Inputs and Outputs:**
- Input: Two non-negative integers $x$ and $y$ representing coordinates in a 2D grid.
- Output: A single 64-bit unsigned integer $m$ representing the 2D Morton index.

**Data Structures:**
- The implementation uses standard 64-bit unsigned integer types (`uint64_t` in C++, `int` in Python 3 which supports arbitrary precision, but logic assumes 64-bit width for masking).

**Steps:**
1. **Spread Bits:** Expand the bits of each coordinate into a 64-bit integer such that the bits of the original coordinate occupy every other bit position (even positions for $x$, odd for $y$). This is done via a sequence of bitwise OR and shift operations with specific masks.
2. **Combine:** Interleave the spread bits of $x$ and $y$ by OR-ing the spread $x$ with the spread $y$ shifted left by 1.

**Formulas:**
Let $S(v)$ be the spread function for a value $v$. The Morton index $m$ is:
$$ m = S(x) \mid (S(y) \ll 1) $$

The spread function $S(v)$ for a 64-bit integer is defined as:
$$
\begin{aligned}
v_1 &= v \ \& \ 0x00000000FFFFFFFF \\
v_2 &= (v_1 \mid (v_1 \ll 32)) \ \& \ 0x0000FFFF0000FFFF \\
v_3 &= (v_2 \mid (v_2 \ll 16)) \ \& \ 0x00FF00FF00FF00FF \\
v_4 &= (v_3 \mid (v_3 \ll 8)) \ \& \ 0x0F0F0F0F0F0F0F0F \\
v_5 &= (v_4 \mid (v_4 \ll 4)) \ \& \ 0x3333333333333333 \\
S(v) &= (v_5 \mid (v_5 \ll 2)) \ \& \ 0x5555555555555555
\end{aligned}
$$

**Parameters:**
- Not applicable for the core encoding function.

**Edge Cases and Pitfalls:**
- **Precision:** The standard implementation assumes 64-bit integers. If coordinates exceed 32 bits, the spread logic must be extended or a larger integer type used.
- **Boundaries:** The method assumes non-negative coordinates. Negative coordinates require two's complement handling or offsetting before encoding.
- **Degenerate Input:** Input values must be integers. Floating-point coordinates must be discretized (e.g., multiplied by a scale factor and cast to integer) before encoding.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spread2(v: int) -> int:
    """Spread bits of v into even positions of a 64-bit integer."""
    v &= 0x00000000FFFFFFFF
    v = (v | (v << 32)) & 0x0000FFFF0000FFFF
    v = (v | (v << 16)) & 0x00FF00FF00FF00FF
    v = (v | (v << 8)) & 0x0F0F0F0F0F0F0F0F
    v = (v | (v << 4)) & 0x3333333333333333
    v = (v | (v << 2)) & 0x5555555555555555
    return v

def encode_2d_morton(x: int, y: int) -> int:
    """Encode 2D coordinates (x, y) into a 2D Morton index."""
    return spread2(x) | (spread2(y) << 1)
```

## Subtopics

### Overlapping Morton Grid Generation
This variant generates a 2D array of Morton indices and then constructs a list of overlapping windows over this grid. It is used when neighborhood queries require accessing a block of spatially contiguous Morton codes that may span across the boundaries of individual Morton-encoded cells. It is chosen when the application requires sliding window operations over a Morton-ordered dataset. The process involves creating a `morton_grid` where `morton_grid[y][x] = encode_2d_morton(x, y)`, then iterating through the grid with a `window_size` to collect subgrids of coordinates. Parameters include `num_squares_x`, `num_squares_y`, and `window_size` (default 2) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Encoding is $O(1)$ with a fixed number of bitwise operations (constant time relative to coordinate magnitude, assuming fixed-width integers).
- **Memory Complexity:** $O(1)$ for the encoding function itself. Storing the grid requires $O(N \times M)$ space for an $N \times M$ grid.
- **Measured Performance:** Bitwise operations are extremely fast on modern CPUs. The method is significantly faster than Hilbert curve encoding, which requires conditional branching or table lookups.
- **Strengths:** Simple implementation, no lookup tables required, excellent for cache-friendly linearization of 2D arrays.
- **Limitations:** Locality preservation is inferior to Hilbert curves; points adjacent in space may have distant Morton indices if they cross quadrant boundaries.
- **When to use:** Ideal for dense grid indexing, texture mapping, and simple spatial hashing where implementation simplicity and speed are prioritized over perfect locality.
- **When not to use:** When strict locality preservation is critical for performance (e.g., large-scale spatial queries in sparse datasets), consider Hilbert codes or [Morton Code Spatial Sorting](4818-morton-code-spatial-sorting.md) with additional post-processing.

## Background
The 2D Morton code is a specific case of the general Morton code (Z-order curve), named after Glenn Morton. It has been widely used in computer graphics, database indexing, and spatial data structures for decades. The specific bit-interleaving algorithm provided in the sources is a standard efficient implementation for 64-bit integers, popularized in high-performance computing contexts [S1, unofficial]. It is a foundational component in [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md).

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S2] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
