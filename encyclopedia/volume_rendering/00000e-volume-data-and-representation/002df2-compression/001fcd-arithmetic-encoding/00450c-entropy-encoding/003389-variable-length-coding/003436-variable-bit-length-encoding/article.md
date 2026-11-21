# Variable Bit Length Encoding

Also called: 可变字长编码

## Overview
Variable Bit Length Encoding (VBLE) is a lossless compression technique for volume data that reduces storage by encoding voxel values relative to a block's minimum value using a variable number of bits [S1, unofficial]. It solves the inefficiency of fixed-width storage for data with localized low variance by allocating bit depth proportional to the local value range [S1, unofficial]. Unlike its parent [Variable-Length Coding](7012-variable-length-coding.md), which determines bit depth per brick based on the global range of that brick, VBLE typically operates on a per-voxel or per-node basis within a hierarchical structure, often requiring combination with Run-Length Encoding (RLE) to efficiently store the variable bit-length metadata [S1, unofficial].

## How it works
**Inputs:** A volume dataset, typically organized as a sequence of 2D slices.
**Outputs:** A compressed bitstream containing the base values, the variable-length encoded residuals, and the bit-length descriptors.

**Data Structures:**
1.  **Quadtree:** A spatial hierarchy built for each data slice. Each node represents a block of voxels.
2.  **Bitstream:** A sequential stream of bits.
3.  **RLE Table:** A parallel structure storing the run-lengths of bit-length codes to mitigate the overhead of storing individual bit-lengths [S1, unofficial].

**Steps:**
1.  **Quadtree Construction:** For each slice of the volume data, construct a quadtree structure. The leaves of the quadtree correspond to individual voxels or small blocks of voxels [S1, unofficial].
2.  **Base Value Selection:** For each leaf node (or block), identify the minimum voxel value, denoted as $v_{min}$. This serves as the base value for the block [S1, unofficial].
3.  **Residual Calculation:** Calculate the difference (residual) for each voxel $v_i$ in the block:
    $$ r_i = v_i - v_{min} $$
    Since $v_{min}$ is the minimum, $r_i \ge 0$ [S1, unofficial].
4.  **Bit-Length Determination:** Determine the number of bits $n_i$ required to represent each residual $r_i$. This is typically:
    $$ n_i = \lceil \log_2(\max(r_i) + 1) \rceil $$
    Or, if encoding per-voxel variable length, $n_i$ is the minimum bits needed to store $r_i$ specifically. If the block is uniform, $n_i$ is constant for all voxels in that block [S1, unofficial].
5.  **Encoding Bit-Lengths:** Store the bit-length $n_i$ for each voxel or block. Because storing $n_i$ for every voxel adds overhead, this step is usually combined with Run-Length Encoding (RLE) to compress sequences of identical bit-lengths [S1, unofficial].
6.  **Encoding Residuals:** Encode the residuals $r_i$ using exactly $n_i$ bits. The residuals are packed into the bitstream sequentially [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vble_encode_slice(slice_data):
    """
    Encodes a single 2D slice using VBLE with quadtree and RLE.
    slice_data: 2D array of voxel values.
    Returns: bitstream, rle_table
    """
    # 1. Build Quadtree (Simplified: assume leaf nodes are individual voxels for VBLE per-voxel)
    # In practice, a quadtree is built to identify regions of uniform bit-length.
    quadtree = build_quadtree(slice_data)
    
    bitstream = []
    rle_codes = [] # Stores (bit_length, run_length)
    
    # 2. Traverse Quadtree (Pre-order or Leaf-level)
    # For VBLE, we often process leaves.
    leaves = get_leaves(quadtree)
    
    current_bit_length = None
    run_length = 0
    
    for leaf in leaves:
        voxels = leaf.get_voxels()
        if not voxels:
            continue
            
        # 3. Calculate Base and Residuals
        v_min = min(voxels)
        residuals = [v - v_min for v in voxels]
        
        # 4. Determine Bit Length for this block/voxel
        # If leaf is a single voxel, n is bits for that residual.
        # If leaf is a block, n is bits for max residual in block.
        max_residual = max(residuals)
        n = ceil_log2(max_residual + 1)
        
        # 5. RLE Encoding of Bit Lengths
        if n == current_bit_length:
            run_length += 1
        else:
            if current_bit_length is not None:
                rle_codes.append((current_bit_length, run_length))
            current_bit_length = n
            run_length = 1
            
        # 6. Encode Residuals into Bitstream
        for r in residuals:
            # Convert residual to binary string of length n
            bits = format(r, f'0{n}b')
            bitstream.extend(bits)
            
    # Append final RLE entry
    if current_bit_length is not None:
        rle_codes.append((current_bit_length, run_length))
        
    return bitstream, rle_codes

def ceil_log2(x):
    if x <= 0: return 1
    return x.bit_length()
```

**Parameters:**
*   **Quadtree Depth:** Not given in the sources. [G] Typically determined by a threshold on variance or block size.
*   **RLE Threshold:** Not given. [G] Used to decide when to start a new run.

**Edge Cases and Pitfalls:**
*   **Overhead:** VBLE requires storing the bit-length for each voxel or block. Without RLE, this overhead can negate compression gains [S1, unofficial].
*   **Precision:** Residuals must be non-negative. If $v_{min}$ is not the true minimum, negative residuals will cause encoding errors.
*   **Boundary Conditions:** When processing slices, ensure that the quadtree construction handles slice boundaries correctly.

## Subtopics

### [Greedy Merging for VV Codes](17555-greedy-merging-for-vv-codes.md)
Greedy Merging is a heuristic method for constructing Variable-length Variable-length (VV) codes, which are a variant of VBLE. It builds on [Variable Bit Length Encoding](7195-variable-bit-length-encoding.md) by using a local optimization strategy to approximate the optimal parse tree structure. Unlike exhaustive search, it is computationally efficient but is not guaranteed to produce the optimal coding rate. It iteratively merges the least-probable leaf in a parse tree to reduce code length.

### RBUC Decoder Implementation
The RBUC (Recursive Bit-Uniform Coding) decoder uses pre-order traversal of the RBUC tree, replaced by three nested loops (one per level) maintaining 'current' bit length on each level. Loops are unrolled in GPU implementation for conditional-free code. Decoder determines number of bits to read for next few nodes, fetches variable-length residuals from bit stream. Signed residuals are stored biased by $2^{w-1}$ to move into unsigned range $[0, 2^{w-1}]$ [S2].

### RBUC Residual Coding
RBUC is a variable-length lossless coding for residuals. It amortizes length bits over several consecutive residuals using a tree structure. Each interior node stores $w = \max_i(\lceil \log_2(x_i + 1) \rceil)$ over child nodes with unsigned values $x_i$. Residuals are mapped to unsigned ordinals $(0, -1, +1, -2, +2...)$ for computing $w$. Trees are shallow (4 levels for residuals up to 127 bits). Static branching is used: lowest-level branching factor 4-5 vertices, second-level branching ~16 nodes. Trees avoid grouping residuals from end of one row with start of next (spatial locality). Each row partitioned into groups of 3-5 vertices, second-level groups span whole rows [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the quadtree construction and traversal. Building the quadtree is $O(N)$ for $N$ voxels. Encoding is linear in the number of voxels.
*   **Memory Complexity:** Requires storage for the quadtree structure and the RLE table. The compressed bitstream size is proportional to the sum of bit-lengths of residuals.
*   **Performance:** Effective for volume data with smooth regions or low variance blocks. Performance degrades if the data has high entropy or uniform high variance.
*   **Strengths:** Adapts to local data characteristics. Simple to implement.
*   **Limitations:** Overhead of bit-length storage. Requires RLE to be effective. Not suitable for highly random data.
*   **When to use:** Medical imaging data, scientific data with large background regions.
*   **When not to use:** Data with high entropy or no spatial coherence.

## Background
VBLE is a technique used in volume data compression, particularly in medical imaging and scientific visualization. It is often used in conjunction with other techniques like RLE and quadtree structures. The method is described in surveys of volume data compression techniques [S1, unofficial]. RBUC, a variant, was proposed by Moffat and Anh for terrain data decompression and rendering [S2].

## Sources
[S1] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
