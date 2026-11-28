# Variable Bit Length Encoding

Also called: 可变字长编码

## Overview
Variable Bit Length Encoding (VBLE) is a lossless compression technique for volume data that reduces storage by encoding voxel values relative to a block's minimum value using a variable number of bits [S1, unofficial]. It solves the inefficiency of fixed-width storage for data with localized low variance by allocating bit depth proportional to the local value range [S1, unofficial].

## How it works
**Inputs:** A volume dataset, typically organized as a sequence of 2D slices [S1, unofficial].
**Outputs:** A compressed bitstream containing the base values, the variable-length encoded residuals, and the bit-length descriptors [G].

**Data Structures:**
1.  **Quadtree:** A spatial hierarchy built for each data slice [S1, unofficial].
2.  **Bitstream:** A sequential stream of bits [G].
3.  **RLE Table:** A parallel structure storing run-lengths of bit-length codes to mitigate the overhead of storing individual bit-lengths [S1, unofficial].

**Steps:**
1.  **Quadtree Construction:** For each slice of the volume data, construct a quadtree structure [S1, unofficial].
2.  **Base Value Selection:** For each leaf node or block, identify the minimum voxel value, denoted as $v_{min}$ [G].
3.  **Residual Calculation:** Calculate the difference (residual) for each voxel $v_i$ in the block:
    $$ r_i = v_i - v_{min} $$
    [G].
4.  **Bit-Length Determination:** Determine the number of bits $n_i$ required to represent each residual $r_i$. For a block, this is typically:
    $$ n_i = \lceil \log_2(\max(r_i) + 1) \rceil $$
    [G]. For per-voxel encoding, $n_i$ is the minimum bits needed to store $r_i$ specifically [G].
5.  **Encoding Bit-Lengths:** Store the bit-length $n_i$ for each voxel or block. Because storing $n_i$ for every voxel adds overhead, this step is usually combined with Run-Length Encoding (RLE) to compress sequences of identical bit-lengths [S1, unofficial].
6.  **Encoding Residuals:** Encode the residuals $r_i$ using exactly $n_i$ bits [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def vble_encode_slice(slice_data):
    """
    Encodes a single 2D slice using VBLE with quadtree and RLE.
    slice_data: 2D numpy array of voxel values.
    Returns: bitstream (list of ints), rle_table (list of tuples)
    """
    def build_quadtree(data, x, y, w, h):
        if w <= 1 or h <= 1:
            return {'type': 'leaf', 'data': data[y:y+h, x:x+w]}
        mid_x, mid_y = x + w // 2, y + h // 2
        return {'type': 'node', 'children': [
            build_quadtree(data, x, y, w//2, h//2),
            build_quadtree(data, mid_x, y, w//2, h//2),
            build_quadtree(data, x, mid_y, w//2, h//2),
            build_quadtree(data, mid_x, mid_y, w//2, h//2)
        ]}

    def get_leaves(node):
        if node['type'] == 'leaf':
            yield node
        else:
            for child in node['children']:
                yield from get_leaves(child)

    quadtree = build_quadtree(slice_data, 0, 0, slice_data.shape[1], slice_data.shape[0])
    bitstream = []
    rle_codes = []
    current_bit_length = None
    run_length = 0

    for leaf in get_leaves(quadtree):
        voxels = leaf['data'].flatten()
        if len(voxels) == 0:
            continue

        # Step 2: Base Value Selection
        v_min = min(voxels)

        # Step 3: Residual Calculation
        residuals = [v - v_min for v in voxels]

        # Step 4: Bit-Length Determination
        max_residual = max(residuals)
        n = math.ceil(math.log2(max_residual + 1)) if max_residual > 0 else 1

        # Step 5: Encoding Bit-Lengths (RLE)
        if n == current_bit_length:
            run_length += 1
        else:
            if current_bit_length is not None:
                rle_codes.append((current_bit_length, run_length))
            current_bit_length = n
            run_length = 1

        # Step 6: Encoding Residuals
        for r in residuals:
            bits = format(r, f'0{n}b')
            bitstream.extend([int(b) for b in bits])

    if current_bit_length is not None:
        rle_codes.append((current_bit_length, run_length))

    return bitstream, rle_codes
```

**Parameters:**
*   **Quadtree Depth:** Not given in the sources. [G] Typically determined by a threshold on variance or block size.
*   **RLE Threshold:** Not given. [G] Used to decide when to start a new run.

**Edge Cases and Pitfalls:**
*   **Overhead:** VBLE requires storing the bit-length for each voxel or block. Without RLE, this overhead can negate compression gains [S1, unofficial].
*   **Precision:** Residuals must be non-negative. If $v_{min}$ is not the true minimum, negative residuals will cause encoding errors [G].
*   **Boundary Conditions:** When processing slices, ensure that the quadtree construction handles slice boundaries correctly [G].

## Subtopics

### [Greedy Merging for VV Codes](17555-greedy-merging-for-vv-codes.md)
Greedy Merging is a heuristic method for constructing Variable-length Variable-length (VV) codes, which are a variant of VBLE. It builds on [Variable Bit Length Encoding](7195-variable-bit-length-encoding.md) by using a local optimization strategy to approximate the optimal parse tree structure. Unlike exhaustive search, it is computationally efficient but is not guaranteed to produce the optimal coding rate. It iteratively merges the least-probable leaf in a parse tree to reduce code length.

### RBUC Decoder Implementation
The RBUC (Recursive Bit-Uniform Coding) decoder uses pre-order traversal of the RBUC tree, replaced by three nested loops (one per level) maintaining 'current' bit length on each level [S2]. Loops are unrolled in GPU implementation for conditional-free code [S2]. Decoder determines number of bits to read for next few nodes, fetches variable-length residuals from bit stream [S2]. Signed residuals are stored biased by $2^{w-1}$ to move into unsigned range $[0, 2^{w-1}]$ [S2].

### RBUC Residual Coding
RBUC is a variable-length lossless coding for residuals [S2]. It amortizes length bits over several consecutive residuals using a tree structure [S2]. Each interior node stores $w = \max_i(\lceil \log_2(x_i + 1) \rceil)$ over child nodes with unsigned values $x_i$ [S2]. Residuals are mapped to unsigned ordinals $(0, -1, +1, -2, +2...)$ for computing $w$ [S2]. Trees are shallow (4 levels for residuals up to 127 bits) [S2]. Static branching is used: lowest-level branching factor 4-5 vertices, second-level branching ~16 nodes [S2]. Trees avoid grouping residuals from end of one row with start of next (spatial locality) [S2]. Each row partitioned into groups of 3-5 vertices, second-level groups span whole rows [S2].

### [Bounded Integer Sequence Encoding (BISE)](19687-bounded-integer-sequence-encoding-bise.md)
Bounded Integer Sequence Encoding (BISE) is a variable-length coding method that stores sequences of integers from arbitrary alphabets (up to 256 symbols) using a fractional number of bits per symbol. It solves the inefficiency of rounding up symbol counts to the next power of two by packing symbols into groups that align with bit boundaries, specifically using binary, trit (3-symbol), and quint (5-symbol) encodings. Unlike standard binary encoding, BISE allows the compressor to select the most space-efficient alphabet representation for a given symbol count, significantly reducing storage waste for non-power-of-two alphabet sizes.

## Operating characteristics
*   **Time Complexity:** Depends on the quadtree construction and traversal. Building the quadtree is $O(N)$ for $N$ voxels. Encoding is linear in the number of voxels [G].
*   **Memory Complexity:** Requires storage for the quadtree structure and the RLE table. The compressed bitstream size is proportional to the sum of bit-lengths of residuals [G].
*   **Performance:** Effective for volume data with smooth regions or low variance blocks. Performance degrades if the data has high entropy or uniform high variance [G].
*   **Strengths:** Adapts to local data characteristics. Simple to implement [G].
*   **Limitations:** Overhead of bit-length storage. Requires RLE to be effective. Not suitable for highly random data [G].
*   **When to use:** Medical imaging data, scientific data with large background regions [S1, unofficial].
*   **When not to use:** Data with high entropy or no spatial coherence [G].

## Background
VBLE is a technique used in volume data compression, particularly in medical imaging and scientific visualization [S1, unofficial]. It is often used in conjunction with other techniques like RLE and quadtree structures [S1, unofficial]. RBUC, a variant, was proposed by Moffat and Anh for terrain data decompression and rendering [S2].

## Sources
[S1] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
