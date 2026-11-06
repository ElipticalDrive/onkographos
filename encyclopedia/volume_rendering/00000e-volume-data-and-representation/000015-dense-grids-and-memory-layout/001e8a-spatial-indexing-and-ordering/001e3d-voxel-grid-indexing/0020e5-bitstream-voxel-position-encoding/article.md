# Bitstream Voxel Position Encoding
Also called: 3-bit interleaved position encoding
## Overview
Bitstream Voxel Position Encoding is a method to encode the 3D position of a voxel into a linear bitstream for tree traversal, where each child index is determined by 3 bits [S1, unofficial]. It solves the problem of mapping spatial coordinates to a hierarchical structure (such as an Octree) by creating a deterministic bit sequence that guides navigation from the root to the leaf node representing the voxel [S1, unofficial].
## How it works
### Inputs and Outputs
- **Input**: A voxel $v$ with integer coordinates $p = (x, y, z)$ [S1, unofficial].
- **Output**: A bitstream $b$ representing the path to the voxel in the tree [S1, unofficial].

### Data Structures
- **Bitstream $b$**: A linear sequence of bits where specific positions are set based on coordinate bits [S1, unofficial].
- **Voxel Position $p$**: An integer tuple $(x, y, z)$ encoded over a fixed width [S1, unofficial].

### Steps
1. Initialize an empty bitstream $b$ [S1, unofficial].
2. Retrieve the voxel position $p = (x, y, z)$ [S1, unofficial].
3. Iterate with a counter $i$ from 1 to 32 [S1, unofficial].
4. For each $i$:
   - Extract the $i$-th bit of the x-coordinate, denoted $bit(x, i)$ [S1, unofficial].
   - Set the bit at position $3 \cdot i$ in $b$ to $bit(x, i)$ [S1, unofficial].
   - Extract the $i$-th bit of the y-coordinate, denoted $bit(y, i)$ [S1, unofficial].
   - Set the bit at position $3 \cdot i - 1$ in $b$ to $bit(y, i)$ [S1, unofficial].
   - Extract the $i$-th bit of the z-coordinate, denoted $bit(z, i)$ [S1, unofficial].
   - Set the bit at position $3 \cdot i - 2$ in $b$ to $bit(z, i)$ [S1, unofficial].
5. Return the constructed bitstream $b$ [S1, unofficial].

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
- **Negative Coordinates**: The sources explicitly state that no coordinate component should be negative ("para ningún componente de la coordenada sea negativo") [S1, unofficial]. Implementers must ensure coordinates are offset or unsigned before encoding [G].
- **Coordinate Width**: The loop runs to 32, implying 32-bit integer coordinates [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_voxel_bitstream(x: int, y: int, z: int) -> list:
    """
    Encodes voxel coordinates into a bitstream.
    Assumes x, y, z are non-negative integers.
    Returns a list of bits (0 or 1) using 1-based indexing for positions 1-96.
    """
    bitstream = [0] * 97  # Indices 1 to 96 are used
    for i in range(1, 33):
        # Extract i-th bit. Assumes i=1 corresponds to the least significant bit [G].
        bitstream[3 * i] = (x >> (i - 1)) & 1
        bitstream[3 * i - 1] = (y >> (i - 1)) & 1
        bitstream[3 * i - 2] = (z >> (i - 1)) & 1
    return bitstream
```
## Subtopics

### [Packed Voxel Cell Encoding](3281-packed-voxel-cell-encoding.md)
Packed Voxel Cell Encoding is a bitstream method that compresses a voxel's 3D integer coordinate and the axis along which a ray exited that voxel into a single 32-bit unsigned integer. It solves the problem of efficiently returning hit voxel data from a ray marcher while preserving the geometric context required for normal estimation and subsequent traversal. It differs from its parent [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md) by explicitly storing the crossing axis alongside the coordinate, enabling direct reconstruction of the intersection point and surface normal without additional ray-marching steps.

## Operating characteristics
- **Time Complexity**: The encoding requires a fixed number of iterations (32) regardless of the scene size, resulting in $O(1)$ time complexity per voxel [S1, unofficial].
- **Memory Complexity**: The bitstream size is proportional to the number of bits used for coordinates, specifically 96 bits for 32-bit coordinates [S1, unofficial].
- **Strengths**: Simple to implement; deterministic mapping; preserves spatial hierarchy for tree traversal [S1, unofficial].
- **Limitations**: Requires non-negative coordinates [S1, unofficial].
## Background
The method originates from research on Sparse Voxel Octrees (SVO) for global illumination, specifically in the context of compression and efficiency [S1, unofficial]. It was developed to facilitate the traversal and insertion of voxels into an Octree structure by converting spatial positions into a linear bit sequence that directly maps to the tree's branching factor of 8 (3 bits per level) [S1, unofficial]. The technique is described in a thesis or report titled "Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global" [S1, unofficial].
## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
