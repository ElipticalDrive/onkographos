# Hash Grids
Also called: Coordinate Hashing, Vertex Hashing for Voxel Grid
## Overview
Hash Grids map continuous 3D vertex positions to discrete bucket indices in a hash table using a multiplicative hash with prime constants [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: `coords`, a list or array of coordinates supporting up to 7 dimensions [S2, unofficial].
- Input: `log2_hashmap_size`, the logarithm of the hash table size $T$ with respect to 2 [S2, unofficial].
- Output: Hashed bucket index.

**Data Structures:**
- `primes`: An array of prime constants used for the hash function with values `[1, 2654435761, 805459861, 3674653429, 2097192037, 1434869437, 2165219737]` [S2, unofficial].
- `BOX_OFFSETS`: A tensor defining 3D offsets as `[[[i,j,k] for i in [0, 1] for j in [0, 1] for k in [0, 1]]]` [S2, unofficial].

**Formula:**
The hash value is computed using a multiplicative hash with prime constants [S1, unofficial]:
$$
\text{hash} = ((\text{pos}.x \times 73856093) \oplus (\text{pos}.y \times 19349663) \oplus (\text{pos}.z \times 83492791)) \pmod n
$$
where $\oplus$ denotes bitwise XOR and $n$ is the table size [S1, unofficial].

**Pseudocode:**
The implementation body is cut off in the provided text [S2, unofficial]. The following pseudocode implements the available signature, constants, and formula in standard Python [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hash(coords, log2_hashmap_size):
    """
    coords: list or array of coordinates (supports up to 7 dimensions)
    log2_hashmap_size: logarithm of hash table size T w.r.t 2
    """
    primes = [1, 2654435761, 805459861, 3674653429, 2097192037, 1434869437, 2165219737]
    table_size = 1 << log2_hashmap_size

    h = 0
    for i, coord in enumerate(coords):
        h ^= int(coord) * primes[i]

    return h % table_size
```

**Parameters:**
- `log2_hashmap_size`: Typical values are not given in the sources [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- The implementation body is cut off in the provided text [S2, unofficial].
- The source formula uses specific prime constants for $x$, $y$, and $z$, but the general function supports up to 7 dimensions [S2, unofficial].
- The handling of dimensions 4–7 is not specified in the source formula [S2, unofficial].
- Precision issues may arise if coordinate values are large, affecting the modulo operation [G].
## Subtopics

### [Voxel World Data Structure](1101-voxel-world-data-structure.md)
The Voxel World Data Structure is a sparse representation for voxel worlds that stores voxel existence using a hash set of integer-based 3D vectors. It efficiently tracks which grid-aligned positions contain solid geometry without allocating memory for empty space. It differs from its parent [Hash Grids](936-hash-grids.md) by operating on discrete world coordinates for voxel placement rather than hashing continuous vertex positions for feature extraction, and it serves as the foundational sparse storage layer for [Multi-type Voxel Storage](1104-multi-type-voxel-storage.md).

### [Hash Grid Loss](2093-hash-grid-loss.md)
Hash Grid Loss is a regularization term used in compressive neural volume rendering to minimize the bit consumption of the binary occupancy structure of a hash grid during training. It operates by modeling the probability distribution of the binary hash grid values and optimizing the negative log-likelihood, effectively driving the model to create a more concentrated (sparse) hash grid structure. It is distinct from sibling methods like Voxel World Data Structure which manage spatial indexing, and from Entropy Loss which focuses on the continuous or quantized attribute values stored within active cells rather than the occupancy bits themselves.

## Operating characteristics
Time and memory complexity: Not given in the sources [S1, S2, unofficial].
Measured performance: Not given in the sources [S1, S2, unofficial].
Scaling: Not given in the sources [S1, S2, unofficial].
Strengths and limitations: Not given in the sources [S1, S2, unofficial].
When to use it and when not: Not given in the sources [S1, S2, unofficial].
## Background
Origin: Not given in the sources [S1, S2, unofficial].
History and development: Not given in the sources [S1, S2, unofficial].
Where it is used: Not given in the sources [S1, S2, unofficial].
## Sources
[S1] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S2] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
