# Bitstream Voxel Position Encoding
Also called: 3-bit interleaved position encoding
## Overview
Bitstream Voxel Position Encoding is a method to encode the 3D position of a voxel into a linear bitstream for tree traversal, where each child index is determined by 3 bits [S1, unofficial]. It solves the problem of mapping spatial coordinates to a hierarchical structure (such as an Octree) by creating a deterministic bit sequence that guides navigation from the root to the leaf node representing the voxel [S1, unofficial]. It differs from its siblings [Coordinate Space Conversion](1785-coordinate-space-conversion.md) and [World to UVW Normalization](2229-world-to-uvw-normalization.md) by operating directly on integer bit patterns rather than floating-point values, and from [Spatial Hashing](5002-spatial-hashing.md) by preserving the hierarchical spatial locality required for tree-based culling and compression [S1, unofficial].
## How it works
### Inputs and Outputs
- **Input**: A voxel $v$ with integer coordinates $p = (x, y, z)$. The coordinates must be non-negative [S1, unofficial].
- **Output**: A bitstream $b$ representing the path to the voxel in the tree.

### Data Structures
- **Bitstream $b$**: A linear sequence of bits. The sources describe it as a structure where bits are set at specific indices [S1, unofficial].
- **Voxel Position $p$**: An integer tuple $(x, y, z)$. The sources imply a fixed-width representation (32 bits per coordinate) for the encoding loop [S1, unofficial].

### Steps
1. Initialize an empty bitstream $b$.
2. Retrieve the voxel position $p = (x, y, z)$.
3. Iterate with a counter $i$ from 1 to 32.
4. For each $i$:
   - Extract the $i$-th bit of the x-coordinate, denoted $bit(x, i)$.
   - Set the bit at position $3 \cdot i$ in $b$ to $bit(x, i)$.
   - Extract the $i$-th bit of the y-coordinate, denoted $bit(y, i)$.
   - Set the bit at position $3 \cdot i - 1$ in $b$ to $bit(y, i)$.
   - Extract the $i$-th bit of the z-coordinate, denoted $bit(z, i)$.
   - Set the bit at position $3 \cdot i - 2$ in $b$ to $bit(z, i)$.
5. Return the constructed bitstream $b$.

### Formulas
The position of the $i$-th bit of coordinate $c \in \{x, y, z\}$ in the bitstream $b$ is given by:
$$
\text{index}_x(i) = 3 \cdot i
$$
$$
\text{index}_y(i) = 3 \cdot i - 1
$$
$$
\text{index}_z(i) = 3 \cdot i - 2
$$
where $i$ is the loop counter ranging from 1 to 32 [S1, unofficial].

### Parameters
- $i$: The loop counter representing the bit index, ranging from 1 to 32 [S1, unofficial].

### Edge Cases and Pitfalls
- **Negative Coordinates**: The sources explicitly state that no coordinate component should be negative ("para ningún componente de la coordenada sea negativo") [S1, unofficial]. Implementers must ensure coordinates are offset or unsigned before encoding.
- **Bit Indexing**: The sources use 1-based indexing for the loop $i$ and bit extraction [S1, unofficial]. Standard programming languages often use 0-based indexing; adjustments are required when implementing.
- **Coordinate Width**: The loop runs to 32, implying 32-bit integer coordinates. If coordinates exceed this range, the encoding will overflow or truncate [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_voxel_bitstream(x: int, y: int, z: int) -> list:
    """
    Encodes voxel coordinates into a bitstream.
    Assumes x, y, z are non-negative integers.
    Returns a list of bits (0 or 1).
    """
    # Initialize bitstream. Size is 3 bits per coordinate per loop iteration.
    # Loop runs 1 to 32. Max index is 3*32 = 96.
    # We use a list of size 97 to accommodate 1-based indexing for simplicity,
    # or map to 0-based. The source uses 1-based positions.
    # Let's use a dictionary or large list. A list of 97 elements (indices 0-96)
    # where we write to 3*i, 3*i-1, 3*i-2.
    # Note: 3*i-2 for i=1 is 1. So indices 1 to 96 are used.
    bitstream = [0] * 97 
    
    for i in range(1, 33):
        # Extract i-th bit. Assuming bit 1 is the least significant or most significant?
        # The source says "bit(x(p), i)". Usually, in such encodings, 
        # the order determines the traversal (MSB first for top-down).
        # However, the source does not specify bit significance (LSB vs MSB).
        # We assume a standard bit extraction function.
        # bit_n(n, i) returns the i-th bit of n.
        
        bx = get_bit(x, i)
        by = get_bit(y, i)
        bz = get_bit(z, i)
        
        # Set bits in the stream
        # Position 3*i
        bitstream[3 * i] = bx
        # Position 3*i - 1
        bitstream[3 * i - 1] = by
        # Position 3*i - 2
        bitstream[3 * i - 2] = bz
        
    return bitstream

def get_bit(n: int, i: int) -> int:
    """
    Returns the i-th bit of n.
    Source uses 1-based indexing for i.
    Implementation detail [G]: Assuming i=1 corresponds to the least significant bit (LSB)
    or most significant bit (MSB). Without explicit direction, LSB is a common default for
    'i-th bit' in generic contexts, but tree traversal often uses MSB first.
    Given the ambiguity, we implement LSB extraction.
    """
    return (n >> (i - 1)) & 1
```
*Note: The source algorithm uses `group(b, d, 3)` for traversal, which extracts 3 bits from the stream. The encoding order (MSB vs LSB) must match the decoding/traversal logic. The source does not specify if $i=1$ is MSB or LSB. The pseudocode above assumes LSB for `get_bit`, but the traversal `group` function would need to align with this.*

## Subtopics

### [Packed Voxel Cell Encoding](3281-packed-voxel-cell-encoding.md)
Packed Voxel Cell Encoding is a bitstream method that compresses a voxel's 3D integer coordinate and the axis along which a ray exited that voxel into a single 32-bit unsigned integer. It solves the problem of efficiently returning hit voxel data from a ray marcher while preserving the geometric context required for normal estimation and subsequent traversal. It differs from its parent [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md) by explicitly storing the crossing axis alongside the coordinate, enabling direct reconstruction of the intersection point and surface normal without additional ray-marching steps.

## Operating characteristics
- **Time Complexity**: The encoding requires a fixed number of iterations (32) regardless of the scene size, resulting in $O(1)$ time complexity per voxel, assuming fixed-width coordinates [S1, unofficial].
- **Memory Complexity**: The bitstream size is proportional to the number of bits used for coordinates. For 32-bit coordinates, the stream is 96 bits long [S1, unofficial].
- **Strengths**: Simple to implement; deterministic mapping; preserves spatial hierarchy for tree traversal [S1, unofficial].
- **Limitations**: Requires non-negative coordinates; fixed bit-width limits coordinate range; does not compress empty space (unlike the Octree structure it feeds into) [S1, unofficial].
- **When to use**: When building a Sparse Voxel Octree or similar hierarchical structure where fast, deterministic path generation from coordinates is needed [S1, unofficial].
- **When not to use**: When coordinates are negative and cannot be offset; when variable-length encoding is preferred for sparse data.

## Background
The method originates from research on Sparse Voxel Octrees (SVO) for global illumination, specifically in the context of compression and efficiency [S1, unofficial]. It was developed to facilitate the traversal and insertion of voxels into an Octree structure by converting spatial positions into a linear bit sequence that directly maps to the tree's branching factor of 8 (3 bits per level) [S1, unofficial]. The technique is described in a thesis or report titled "Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global" [S1, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
