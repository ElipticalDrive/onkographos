# Z-Order and Hilbert Encoding/Decoding


## Overview
Z-order (Morton) and Hilbert encoding are space-filling curve methods that map $N$-dimensional grid coordinates to a 1D linear index, preserving spatial locality better than row-major strides [S1, S2, unofficial]. Z-order uses bit-interleaving, while Hilbert curves provide superior locality preservation at the cost of higher computational complexity [S1, S2, unofficial]. These methods are primarily used for serializing dense grids, sorting voxel data for cache coherence, and enabling efficient spatial indexing in volume rendering pipelines [S1, S2, S3, unofficial].

## How it works
**Inputs:** 3D integer coordinates $(x, y, z)$ within a bounded grid, and a target bit-depth $B$ (typically 32 or 64 bits total).
**Outputs:** A 1D integer index $I$.

### Data Structures
The core data structure is a fixed-width integer (e.g., `uint32_t` or `uint64_t`) where bits are rearranged from the input coordinates.
- **Z-Order:** Bits are interleaved directly. For 3D, if coordinates are $x = x_{B-1}...x_0$, $y = y_{B-1}...y_0$, $z = z_{B-1}...z_0$, the code is $I = x_{B-1}y_{B-1}z_{B-1}...x_0y_0z_0$.
- **Hilbert:** Bits are transformed via Gray codes and recursive quadrant rotations before interleaving.

### Z-Order Encoding (Morton Code)
1.  **Bit Interleaving:** Extract bits from $x, y, z$ and pack them into $I$.
    $$ I = \sum_{k=0}^{B/3-1} (x_k \ll 2k) | (y_k \ll (2k+1)) | (z_k \ll (2k+2)) $$
    where $x_k$ is the $k$-th bit of $x$.
2.  **Implementation:** Use bitwise operations. For efficiency, parallel bit interleaving algorithms (e.g., using magic constants or lookup tables) are preferred over loops [S1, unofficial].

### Hilbert Encoding
1.  **Gray Code Conversion:** Convert binary coordinates to Gray code to ensure adjacent values differ by one bit, a prerequisite for the Hilbert curve's continuity.
2.  **Quadrant Determination:** Determine the Hilbert curve quadrant ($0-7$ for 3D) based on the most significant bits of the Gray-coded coordinates.
3.  **Rotation/Reflection:** Apply bitwise rotations or reflections to the coordinates based on the quadrant to align the curve direction.
4.  **Recursive Interleaving:** Interleave the transformed bits.
    *Note: Exact bit-manipulation formulas for Hilbert curves are complex and vary by implementation (e.g., `hilbert_encode_cpu` vs `hilbert_encode_cuda` in [S1, unofficial]). The general principle involves mapping $(x,y,z)$ to a path length along the curve.*

### Pseudocode (Z-Order)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def z_order_encode(x: int, y: int, z: int, bits: int = 10) -> int:
    """
    Encodes 3D coordinates into a Z-order (Morton) code.
    bits: number of bits per coordinate (total code size = 3 * bits).
    """
    # Ensure coordinates fit within bits
    mask = (1 << bits) - 1
    x &= mask
    y &= mask
    z &= mask

    code = 0
    for i in range(bits):
        # Interleave bits: x at pos 2i, y at 2i+1, z at 2i+2
        code |= ((x >> i) & 1) << (2 * i)
        code |= ((y >> i) & 1) << (2 * i + 1)
        code |= ((z >> i) & 1) << (2 * i + 2)
    return code

def z_order_decode(code: int, bits: int = 10):
    """
    Decodes a Z-order code back to 3D coordinates.
    """
    mask = (1 << bits) - 1
    x = 0
    y = 0
    z = 0
    for i in range(bits):
        x |= ((code >> (2 * i)) & 1) << i
        y |= ((code >> (2 * i + 1)) & 1) << i
        z |= ((code >> (2 * i + 2)) & 1) << i
    return x, y, z
```

### Parameters and Edge Cases
- **Bit Depth:** Must be sufficient to represent the grid dimensions. If grid size $> 2^{B/3}$, overflow occurs [S1, unofficial].
- **Precision:** Floating-point coordinates must be quantized to integer grid indices before encoding [S1, unofficial].
- **Boundary:** Coordinates must be non-negative and within $[0, 2^B-1]$.

## Subtopics

### Morton Code Sorting for Brick Layout
Uses Morton codes to sort sub-bricks within a parent brick buffer to maintain spatial coherence [S2, unofficial].
- **Method:** An enumeration sort is used where each thread calculates the sorted index of one sub-brick based on its Morton code [S2, unofficial].
- **Constraint:** Maximum 64 sub-bricks per parent brick [S2, unofficial].
- **Benefit:** Ensures nearby bricks in space are adjacent in memory, improving cache coherency during rendering [S2, unofficial].

### Serialized-Morton/ASTC
A variant that rearranges voxel attributes into serialized correlations to enable reuse of ASTC (Adaptive Scalable Texture Compression) codecs [S3, unofficial].
- **Trade-off:** While it removes 2D layout overhead, rearranging attributes can weaken spatial correlations available to UV mapping alone [S3, unofficial].
- **Cost:** Requires additional memory for addressing (e.g., DAG) and introduces latency for random access decoding [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Z-order encoding/decoding is $O(B)$ where $B$ is bit depth, often optimized to $O(1)$ with parallel bit hacks [S1, unofficial]. Hilbert encoding is $O(B)$ but with higher constant factors due to rotation/reflection logic [S1, S2, unofficial].
- **Memory:** Output is a single integer per voxel. No auxiliary structures required for the encoding itself [S1, unofficial].
- **Performance:** Z-order is faster to compute; Hilbert provides better locality for large distances [S1, S2, unofficial].
- **Use Case:** Use Z-order for simple spatial hashing or when computational speed is critical. Use Hilbert when cache performance for large-scale traversal is paramount [S1, S2, unofficial].

## Background
Z-order (Morton order) was introduced by G. M. Morton in 1966 [G]. Hilbert curves were derived from the Hilbert space-filling curve [G]. In volume rendering, these methods are used to linearize 3D grids for GPU memory access patterns [S1, S2, unofficial]. The provided implementations (`o-voxel-gpu`) support both CPU and CUDA execution for high-performance serialization [S1, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S3] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
