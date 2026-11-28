# 2D Morton Code

Also called: 2D Morton Index, 2D Z-order

## Overview
The 2D Morton Code maps two-dimensional grid coordinates $(x, y)$ into a one-dimensional integer index by interleaving their binary representations [S1, unofficial], building on [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md). It preserves spatial locality for dense grids and differs from 3D Morton codes by operating on two dimensions and from Hilbert curves by using direct bit interleaving rather than recursive traversal [G].

## How it works
**Inputs and Outputs:**
- Input: Two non-negative integers $x$ and $y$ representing coordinates in a 2D grid [S1, unofficial].
- Output: A single 64-bit unsigned integer $m$ representing the 2D Morton index [S1, unofficial].

**Data Structures:**
- The implementation uses standard 64-bit unsigned integer types for storage and arithmetic [G].

**Steps:**
1. **Spread Bits:** Expand the bits of each coordinate into a 64-bit integer such that the bits of the original coordinate occupy every other bit position (even positions for $x$, odd for $y$) [S1, unofficial]. This is done via a sequence of bitwise OR and shift operations with specific masks [S1, unofficial].
2. **Combine:** Interleave the spread bits of $x$ and $y$ by OR-ing the spread $x$ with the spread $y$ shifted left by 1 [S1, unofficial].

**Formulas:**
Let $S(v)$ be the spread function for a value $v$ [S1, unofficial]. The Morton index $m$ is:
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
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** The standard implementation assumes 64-bit integers; coordinates exceeding 32 bits require extended spread logic or larger integer types [G].
- **Boundaries:** The method assumes non-negative coordinates; negative coordinates require two's complement handling or coordinate offsetting before encoding [G].
- **Degenerate Input:** Input values must be integers; floating-point coordinates must be discretized (e.g., multiplied by a scale factor and cast to integer) before encoding [G].

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
This variant generates a 2D array of Morton indices and constructs a list of overlapping windows over the grid [S2, unofficial]. It is used when neighborhood queries require accessing a block of spatially contiguous Morton codes that may span across the boundaries of individual Morton-encoded cells [G]. It is chosen when the application requires sliding window operations over a Morton-ordered dataset [G]. The process involves creating a `morton_grid` where `morton_grid[y][x] = encode_2d_morton(x, y)`, then iterating through the grid with a `window_size` to collect subgrids of decoded coordinates [S2, unofficial]. Parameters include `num_squares_x`, `num_squares_y`, and `window_size` (default 2) [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_2d_morton(m: int) -> tuple:
    """Decode a 2D Morton index back to (x, y) coordinates [G]."""
    x = m & 0x5555555555555555
    y = (m & 0xAAAAAAAAAAAAAAAA) >> 1
    x = (x | (x >> 1)) & 0x3333333333333333
    y = (y | (y >> 1)) & 0x3333333333333333
    x = (x | (x >> 2)) & 0x0F0F0F0F0F0F0F0F
    y = (y | (y >> 2)) & 0x0F0F0F0F0F0F0F0F
    x = (x | (x >> 4)) & 0x00FF00FF00FF00FF
    y = (y | (y >> 4)) & 0x00FF00FF00FF00FF
    x = (x | (x >> 8)) & 0x0000FFFF0000FFFF
    y = (y | (y >> 8)) & 0x0000FFFF0000FFFF
    x = (x | (x >> 16)) & 0x00000000FFFFFFFF
    y = (y | (y >> 16)) & 0x00000000FFFFFFFF
    return x, y

def generate_overlapping_morton_grid(num_squares_x: int, num_squares_y: int, window_size: int = 2) -> list:
    morton_grid = [[encode_2d_morton(x, y) for x in range(num_squares_x)] for y in range(num_squares_y)]
    overlapping_grid = []
    for i in range(num_squares_y - window_size + 1):
        for j in range(num_squares_x - window_size + 1):
            window = []
            for dy in range(window_size):
                for dx in range(window_size):
                    y, x = i + dy, j + dx
                    coords = decode_2d_morton(morton_grid[y][x])
                    window.append(coords)
            overlapping_grid.append(window)
    return overlapping_grid
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The 2D Morton code is a specific case of the general Morton code (Z-order curve) [G]. It is a foundational component in [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md) [G].

## Sources
[S1] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S2] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
