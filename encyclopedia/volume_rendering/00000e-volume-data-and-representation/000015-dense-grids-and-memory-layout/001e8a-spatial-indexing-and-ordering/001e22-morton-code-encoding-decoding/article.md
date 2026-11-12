# Morton Code Encoding/Decoding

Also called: MSFC, Morton Coordinate Construction, Morton Order Indexing, Z-order curve, Z-order curve encoding, Z-order encoding

## Overview
Morton Code Encoding/Decoding is a method to rearrange pixels or voxels from an $n$-dimensional regular grid into a linear 1D form using the Morton Space Filling Curve (MSFC), also known as the Z-order curve [S1, S8]. It converts multi-dimensional integer coordinates into a single 1D Morton index by interleaving their binary representations, thereby preserving spatial locality such that geometrically adjacent points have close linear indices [S1][S2, S9, unofficial]. This optimization reduces the number of disk seek operations required to access subsets of data with similar properties, as indices are sorted by value and similar data points are placed contiguously in memory or on disk [S3, unofficial]. It differs from arithmetic strides (row-major) by providing superior cache performance for non-contiguous access patterns, and from Hilbert encoding by using direct bit-interleaving rather than recursive curve traversal [S1, S8].

## How it works
**Inputs and Outputs:**
*   **Encoding:** Input is an $n$-dimensional coordinate vector $(c_0, c_1, \dots, c_{n-1})$ of integers. Output is a single unsigned integer (Morton code) $M$.
*   **Decoding:** Input is a Morton code $M$. Output is the $n$-dimensional coordinate vector $(c_0, c_1, \dots, c_{n-1})$.

**Data Structures:**
*   **Naive Implementation:** No auxiliary structures. Operates directly on integer types (typically `uint64_t` for 3D).
*   **Lookup Table Implementation (for performance):**
    *   `Table`: A flat array of size `dimensions * tableSize`. `Table[dim * tableSize + val]` stores the precomputed interleaved bits for coordinate value `val` in dimension `dim`.
    *   `Magic`: An array of `uint64` constants used to isolate and shift bits during decoding without loops.

**Numbered Steps:**

1.  **Encoding (Naive Bit Interleaving):**
    For each bit index $i$ from 0 to $B-1$ (where $B$ is the number of bits in the coordinate type, e.g., 31 for signed 32-bit, or limited by the output type width):
    *   Extract the $i$-th bit of each coordinate component $c_k$.
    *   Shift the bit to its position in the Morton code. For 3D, the bit from $x$ goes to position $3i$, $y$ to $3i+1$, and $z$ to $3i+2$.
    *   Combine using bitwise OR.

2.  **Decoding (Naive Bit Extraction):**
    For each bit index $i$ from 0 to $\lfloor (W/n) - 1 \rfloor$ (where $W$ is the width of the Morton code type in bits, and $n$ is the dimension):
    *   Extract the bit at position $ni + k$ from $M$ for each dimension $k \in [0, n-1]$.
    *   Shift the extracted bit back to position $i$ in the coordinate component $c_k$.
    *   Combine using bitwise OR.

3.  **Encoding (Lookup Table Method):**
    *   Initialize result $M = 0$.
    *   For each dimension $k$ from 0 to $n-1$:
        *   Retrieve precomputed value $V = \text{Table}[k \times \text{tableSize} + c_k]$.
        *   Update $M = M \mid V$.

4.  **Decoding (Magic Bits Method):**
    *   For each dimension $k$ from 0 to $n-1$:
        *   Initialize $r = (M \gg k) \& \text{Magic}[0]$.
        *   For $j$ from 0 to $\text{len}(\text{Magic})-2$:
            *   $r = (r \oplus (r \gg ((n-1) \times (1 \ll j)))) \& \text{Magic}[j+1]$.
        *   Set $c_k = \text{uint32}(r)$.

**Formulas:**

*   **Naive 3D Encoding:**
    $$ M = \bigvee_{i=0}^{31} \left( (x \& (1 \ll i)) \ll 2i \right) \lor \left( (y \& (1 \ll i)) \ll (2i+1) \right) \lor \left( (z \& (1 \ll i)) \ll (2i+2) \right) $$
    Where $\lor$ is bitwise OR, $\&$ is bitwise AND, and $\ll$ is left shift [S9, unofficial].

*   **Naive 3D Decoding:**
    $$ x = \bigvee_{i=0}^{20} \left( (M \& (1 \ll (3i))) \gg i \right) $$
    $$ y = \bigvee_{i=0}^{20} \left( (M \& (1 \ll (3i+1))) \gg i \right) $$
    $$ z = \bigvee_{i=0}^{20} \left( (M \& (1 \ll (3i+2))) \gg i \right) $$
    Where the loop limit is $\lfloor 64/3 \rfloor = 20$ for 64-bit codes [S9, unofficial].

*   **Magic Bits Decoding (General N-D):**
    $$ r_k = \text{IterativeMask}\left( (M \gg k) \& M_0, M_1, \dots, M_{L-1} \right) $$
    Where $M_j$ are the magic constants and the iteration performs parallel bit extraction [S5, unofficial].

**Parameters:**
*   **Dimensions ($n$):** Number of coordinate components. Supported values are typically 2, 3, and 4, limited by the width of the unsigned integer type (e.g., `uint64_t` supports up to 6 dimensions theoretically, but 3 is standard for voxels) [S5, unofficial].
*   **Table Size:** The maximum value for any single coordinate component that can be encoded. For 3D with `uint64`, the max coordinate is $2^{20}-1$ if the table size is $2^{20}$ [S5, unofficial].
*   **Coordinate Range:** Coordinates must be non-negative integers. In specific implementations (e.g., CUDA kernels), coordinates may be constrained to $[0, 128)$ for `int32` tensors [S2, S7, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def morton_encode_naive(x: int, y: int, z: int) -> int:
    """
    Encodes 3D integer coordinates into a 64-bit Morton code using naive bit interleaving.
    Assumes x, y, z are non-negative and fit within 21 bits each.
    """
    answer = 0
    # Iterate through bits. 64 bits / 3 dimensions = 21 bits per coordinate max.
    for i in range(21):
        # Extract i-th bit of x, y, z
        bit_x = (x >> i) & 1
        bit_y = (y >> i) & 1
        bit_z = (z >> i) & 1
        
        # Place bits in Morton code at positions 3i, 3i+1, 3i+2
        answer |= (bit_x << (3 * i))
        answer |= (bit_y << (3 * i + 1))
        answer |= (bit_z << (3 * i + 2))
        
    return answer

def morton_decode_naive(morton: int) -> tuple:
    """
    Decodes a 64-bit Morton code back into 3D integer coordinates.
    """
    x = 0
    y = 0
    z = 0
    
    # Iterate through bits. 64 bits / 3 dimensions = 21 bits per coordinate max.
    for i in range(21):
        # Extract bits from Morton code
        bit_x = (morton >> (3 * i)) & 1
        bit_y = (morton >> (3 * i + 1)) & 1
        bit_z = (morton >> (3 * i + 2)) & 1
        
        # Place bits back into coordinates
        x |= (bit_x << i)
        y |= (bit_y << i)
        z |= (bit_z << i)
        
    return (x, y, z)

def morton_encode_lookup(vector: list, table: list, table_size: int) -> int:
    """
    Encodes N-dimensional coordinates using a precomputed lookup table.
    vector: list of integers representing coordinates [c0, c1, ...]
    table: flat list where table[dim * table_size + val] contains interleaved bits
    table_size: maximum value for any coordinate component
    """
    result = 0
    dimensions = len(vector)
    
    for k, v in enumerate(vector):
        if v >= table_size:
            raise ValueError(f"Coordinate {k} value {v} exceeds table size {table_size}")
        # Retrieve precomputed interleaved bits for this dimension and value
        result |= table[k * table_size + v]
        
    return result

def morton_decode_magic(code: int, magic: list, dimensions: int) -> list:
    """
    Decodes a Morton code using magic bit constants.
    code: the Morton code integer
    magic: list of uint64 magic constants [M0, M1, ...]
    dimensions: number of dimensions (n)
    """
    result = [0] * dimensions
    d = dimensions
    
    for i in range(d):
        # Extract bits for dimension i
        r = (code >> i) & magic[0]
        
        # Iteratively mask to separate interleaved bits
        for j in range(len(magic) - 1):
            # Shift right by (n-1) * 2^j
            shift = (d - 1) * (1 << j)
            r = (r ^ (r >> shift)) & magic[j + 1]
            
        result[i] = r
        
    return result
```

**Edge Cases and Pitfalls:**
*   **Overflow:** The Morton code width is limited by the integer type. For 3D coordinates using `uint64_t`, the maximum coordinate value is $2^{20}-1$ (since $3 \times 21 = 63$ bits used, leaving 1 bit for sign/overflow if signed, or 21 bits if unsigned). Exceeding this causes bit collision [S5, S9, unofficial].
*   **Input Validation:** Encoding must validate that input vector length does not exceed dimensions and that coordinate values do not exceed the lookup table size (if using LUT) [S5, unofficial].
*   **Decoding Edge Case:** If dimensions is 0 or magic array is empty, decoding returns nil/empty [S5, unofficial].
*   **Signed Integers:** Morton codes are typically unsigned. If coordinates are signed, they must be offset or converted to unsigned before encoding [S9, unofficial].

## Subtopics

### [Morton Index Calculation via Byte-Lookup Table](2384-morton-index-calculation-via-byte-lookup-table.md)
A method to compute the Morton index for a 3D voxel position $(x, y, z)$ by using precomputed lookup tables for each byte of the coordinates, avoiding expensive bitwise interleaving operations. It sets itself apart from naive bitwise interleaving by trading memory access for computational simplicity, and from general Morton encoding by optimizing specifically for chunked voxel data with a hard limit of $256^3$ voxels per chunk. It builds on [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md).

### [Morton Code Generation for Octrees](4673-morton-code-generation-for-octrees.md)
This method generates unique hierarchical spatial keys for octree cells by recursively concatenating binary octant indices to a parent key. It solves the problem of identifying cells both spatially and hierarchically within a dense or sparse octree structure, enabling efficient neighbor and parent lookups. Unlike standard Morton encoding which interleaves global coordinates, this approach builds keys based on the tree's traversal path, where each level adds exactly three bits to the key length.

### [2D Morton Code](4680-2d-morton-code.md)
The 2D Morton Code maps two-dimensional grid coordinates $(x, y)$ into a one-dimensional integer index by interleaving their binary representations, building on [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md). It preserves spatial locality for dense grids and differs from 3D Morton codes by operating on two dimensions and from Hilbert curves by using direct bit interleaving rather than recursive traversal [G].

### [Morton Code Spatial Sorting](4818-morton-code-spatial-sorting.md)
Morton Code Spatial Sorting reorders a set of 3D primitives (such as Gaussian splats or voxel blocks) by their spatial coordinates using a one-dimensional Morton code (Z-order curve) to maximize spatial locality in memory. This ordering ensures that geometrically adjacent primitives are stored contiguously, which significantly improves CPU cache hit rates and reduces memory bandwidth pressure during traversal. It sets itself apart from pure random shuffling (which destroys locality) and pure shuffled Morton orders (which balance locality with hardware load balancing) by prioritizing strict spatial coherence for compression and cache efficiency.

### [Morton Order Indexing for Out-of-Core Rendering](6826-morton-order-indexing-for-out-of-core-rendering.md)
Morton Order Indexing for Out-of-Core Rendering is a data organization scheme that sorts voxel indices based on spatial locality using a recursive Z-shaped curve traversal, reducing disk seek operations by storing similar color/function values contiguously. It differs from standard Morton Code Encoding by integrating the spatial index with a histogram-based look-up table to manage variable-sized voxelsets on disk, enabling efficient out-of-core retrieval.

### [Morton Packing for Texture Atlases](6996-morton-packing-for-texture-atlases.md)
Morton Packing for Texture Atlases is a strategy to pack block-level charts into a 2D atlas using Morton ordering to maximize spatio-temporal coherence. It solves the problem of efficiently encoding high-quality textures alongside compressed geometry in sparse volume data structures where the set of occupied blocks varies per frame. Unlike methods that sacrifice spatial coherence for efficiency or require expensive temporal tracking, this approach preserves 3D spatial structure in the 2D atlas by mapping sorted 3D Morton codes to 2D Morton coordinates.

### Morton Code Encoding/Decoding for Voxel Corners
This variant converts all voxel corners to Morton codes to create a unique list of points to solve, sorts the list, and removes duplicates to identify unique corners [S10, unofficial]. For every encoded unique point, it finds all neighbors and uses a simplification algorithm to solve a linear system, storing the result in an array indexed by the Morton code [S10, unofficial]. During decoding, it iterates over all voxels, finds Morton codes for all corners, and uses binary search in the Morton codes array to find the index for the corner, then looks up the result array to retrieve the calculated distance value [S10, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Naive Encoding/Decoding: $O(B)$, where $B$ is the number of bits in the coordinate (e.g., 21 iterations for 3D `uint64`).
    *   Lookup Table Encoding: $O(n)$, where $n$ is the number of dimensions, assuming $O(1)$ table lookup.
    *   Magic Bits Decoding: $O(n \cdot \log B)$, where the number of magic bit operations is logarithmic with respect to the bit width.
*   **Memory Complexity:**
    *   Naive: $O(1)$ auxiliary space.
    *   Lookup Table: $O(n \cdot S)$, where $S$ is the table size (max coordinate value). For 3D `uint64` with $S=2^{20}$, this is significant ($3 \times 2^{20} \times 8$ bytes $\approx 48$ MB).
*   **Measured Performance:**
    *   Naive bit interleaving is simple but slow for large datasets due to loop overhead.
    *   Lookup tables and magic bits are significantly faster, trading memory for speed [S5, unofficial].
    *   Morton ordering reduces disk seek operations to a worst-case of 8 seeks for accessing any rectangular $n \times n$ subset in a 3D octree, usually fewer [S3, unofficial].
*   **Strengths:**
    *   Preserves spatial locality, improving cache performance for volume rendering and ray tracing.
    *   Hierarchical: Morton order for a higher tree level corresponds to the order of the coarser version of the grid [S8].
    *   Simple to implement (naive version).
*   **Limitations:**
    *   Bit interleaving is not a perfect space-filling curve (Hilbert is better for locality but more complex).
    *   Limited by integer width (e.g., 64-bit limits 3D coordinates to 21 bits).
    *   Lookup table method requires significant memory for large coordinate ranges.
*   **When to Use:**
    *   When spatial locality is critical for cache performance (e.g., sparse voxel octrees, GPU ray tracing).
    *   When out-of-core rendering requires minimizing disk seeks.
*   **When Not to Use:**
    *   When coordinates exceed the bit-width limits of the chosen integer type.
    *   When Hilbert curve's superior locality is required and performance cost is acceptable.

## Background
Morton order was initially proposed by G. M. Morton in 1966 [S1, S8]. It is a linearization of an $n$-dimensional grid corresponding to the order of leaf nodes encountered during a post-order depth-first traversal of the corresponding $2^n$-tree (quadtree in 2D, octree in 3D) [S8]. The Morton code of a cell is its index in this order, obtained by hierarchically subdividing a Z-shaped space-filling curve [S8]. It is widely used in computer graphics for spatial indexing, particularly in sparse voxel octrees and out-of-core rendering systems to optimize data access patterns [S1, S8][S9, unofficial].

## Sources
[S1] n-Dimensional Sparse Voxel Trees for Compact Representation of Dimensionally Upscaled Voxelized Three-Dimensional Scenes, https://doi.org/10.12700/aph.22.9.2025.9.1, fetched 2026-10-27
[S2] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S3] Interactive Visualization of Very Large Datasets Using an Out-of-Core Point-based Approach, https://core.ac.uk/download/484268453.pdf, fetched 2026-10-27 (unofficial)
[S4] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
[S5] GitHub - Jsewill/morton: A Morton Order (Z-Order Curve) library ..., https://github.com/Jsewill/morton, fetched 2026-10-27 (unofficial)
[S6] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S7] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S8] Out-of-Core Construction of Sparse Voxel Octrees, https://graphics.cs.kuleuven.be/publications/BLD13OCCSVO/BLD13OCCSVO_paper.pdf, fetched 2026-10-27
[S9] Out-Of-Core construction of Sparse Voxel Octrees - Jeroen Baert's Blog, https://www.forceflow.be/2012/07/24/out-of-core-construction-of-sparse-voxel-octrees/, fetched 2026-10-30 (unofficial)
[S10] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
