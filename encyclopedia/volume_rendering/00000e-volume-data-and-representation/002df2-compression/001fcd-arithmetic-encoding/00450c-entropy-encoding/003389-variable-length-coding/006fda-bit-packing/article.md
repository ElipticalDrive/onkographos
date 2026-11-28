# Bit Packing

Also called: Packing multiple cells into a single byte

## Overview
Bit packing reduces memory footprint by storing multiple voxel values into a single byte, utilizing fewer bits per voxel than a standard byte (char) [S1, unofficial]. It solves the problem of excessive memory usage in dense voxel chunks by providing linear compression over the full chunk [S1, unofficial].

## How it works
**Inputs:** A chunk of voxel data (e.g., an $8 \times 8 \times 8$ grid) where each voxel value fits within $n$ bits, where $n < 8$ [S1, unofficial].
**Outputs:** A compressed byte array where each byte contains $\lfloor 8/n \rfloor$ voxel values [S1, unofficial].
**Data Structures:**
- Input: Array of $N$ voxels, each represented by an integer requiring $n$ bits.
- Output: Array of $M$ bytes, where $M = \lceil N \times n / 8 \rceil$ [G].

**Steps:**
1. Determine the minimum number of bits $n$ required to represent the range of values in the chunk [G].
2. Calculate the number of voxels per byte: $k = \lfloor 8/n \rfloor$ [G].
3. Iterate through the input voxel array in steps of $k$ [G].
4. For each group of $k$ voxels, pack them into a single byte:
   - Initialize a byte `b` to 0.
   - For $i$ from 0 to $k-1$:
     - Shift `b` left by $n$ bits.
     - OR `b` with the $i$-th voxel value (masked to $n$ bits) [G].
5. If the total number of voxels is not a multiple of $k$, handle the remaining voxels by packing them into the final byte, padding the unused bits with zeros [G].

**Formula:**
For a chunk of size $V$ voxels with $n$ bits per voxel, the compressed size $S$ in bytes is:
$$ S = \left\lceil \frac{V \times n}{8} \right\rceil $$
This matches the example of an $8 \times 8 \times 8$ chunk (512 voxels) with 4-bit packing yielding 256 bytes [S1, unofficial].

**Parameters:**
- $n$: Bits per voxel. Typical values are 4, 5, 6, or 7, depending on the data range [G].
- Chunk size: Commonly $8 \times 8 \times 8$ (512 voxels) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Ensure voxel values are truncated or masked to $n$ bits before packing to avoid overflow into adjacent voxels [G].
- **Boundaries:** If $V \times n$ is not divisible by 8, the last byte contains unused bits. These must be handled consistently during unpacking [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bit_pack(voxels: list[int], bits_per_voxel: int) -> list[int]:
    """
    Pack a list of voxel integers into a list of bytes.
    
    Args:
        voxels: List of integer voxel values.
        bits_per_voxel: Number of bits used per voxel (must be < 8).
        
    Returns:
        List of integers (bytes) representing the packed data.
    """
    if bits_per_voxel >= 8:
        raise ValueError("Bits per voxel must be less than 8 for compression.")
    
    voxels_per_byte = 8 // bits_per_voxel
    packed = []
    
    # Process full groups
    for i in range(0, len(voxels), voxels_per_byte):
        byte_val = 0
        for j in range(voxels_per_byte):
            if i + j < len(voxels):
                voxel_val = voxels[i + j] & ((1 << bits_per_voxel) - 1)
                byte_val = (byte_val << bits_per_voxel) | voxel_val
            else:
                # Pad remaining bits with 0 if chunk is not full
                byte_val = byte_val << bits_per_voxel
        packed.append(byte_val)
        
    return packed
```

## Subtopics

### Generic Leaf Compression (RLE/Zip/Bit Quantization)
Generic leaf compression applies lossless RLE, Zip, or lossy bit quantization directly to the `mLeafDAT` for cases with an arbitrary number of inactive voxel values, rather than just one or two [S2, unofficial]. These schemes can be applied to special cases but are orders of magnitude slower than bit-mask-based codes like bit packing [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ for packing and unpacking, where $N$ is the number of voxels [G].
- Memory: Reduces storage by a factor of $8/n$ [S1, unofficial].

**Measured Performance:**
- For an $8 \times 8 \times 8$ chunk (512 voxels) with 4-bit packing, the size reduces from 512 bytes to 256 bytes [S1, unofficial].

**Scaling:**
- Compression is linear with respect to the chunk size [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Simple implementation, effective for dense data with small ranges [S1, unofficial].
- Limitations: Ineffective if $n \ge 8$ or if data distribution is highly variable requiring large $n$ [G].

**When to Use:**
- Use when voxel values have a small, known range and memory bandwidth is a bottleneck [G].

## Background
Bit packing was discussed as a practical optimization for reducing memory usage in voxel engine chunk-based world storage [S1, unofficial].

## Sources
[S1] Method to compress chunks for voxel engine?, https://www.reddit.com/r/VoxelGameDev/comments/15bycu3/method_to_compress_chunks_for_voxel_engine/, fetched 2026-10-27 (unofficial)
[S2] VDB: High-Resolution Sparse Volumes with Dynamic Topology (Academy Award Winner) [pdf], http://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf, fetched 2026-10-26 (unofficial)
