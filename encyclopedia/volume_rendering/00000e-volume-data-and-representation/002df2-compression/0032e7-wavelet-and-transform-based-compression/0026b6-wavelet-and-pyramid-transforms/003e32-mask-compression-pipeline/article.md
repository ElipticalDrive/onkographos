# Mask Compression Pipeline

Also called: Learnable Mask Compression, Mask Encoding Pipeline

## Overview
The Mask Compression Pipeline is a multi-stage encoding strategy designed to minimize the storage footprint of binarized learnable masks used in neural field representations (such as TensoRF and TriPlane) [S1, S2]. It solves the problem of high memory consumption caused by storing sparse binary masks by exploiting the statistical properties of these masks (high sparsity, block-wise structure) through 8-bit casting, Run-Length Encoding (RLE), and Huffman coding [S1]. It distinguishes itself from general volume compression methods by operating specifically on the *mask* data structure rather than the volumetric coefficients themselves, and by utilizing level-wise separation to handle the varying sparsity patterns across different Discrete Wavelet Transform (DWT) decomposition levels [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A binarized learnable mask $M$ (values 0 or 1) derived from a trained neural field model. The mask may be 2D (for plane-based representations) or 3D, depending on the underlying architecture [S1, S2].
*   **Output:** A compressed bitstream representing the mask.

**Data Structures:**
*   **Mask ($M$):** A binary array where $M_{i,j,k} \in \{0, 1\}$. In the context of DWT-based methods, the mask is associated with specific subbands (LL, LH, HL, HH) at various decomposition levels [S1, S2].
*   **RLE Stream:** A sequence of pairs $(value, count)$ representing consecutive identical elements [S1].
*   **Huffman Tree:** A binary tree mapping frequent RLE values to shorter bit sequences [S1].

**Parameters:**
*   $\lambda_m$: The regularization weight applied during training to control mask sparsity. Lower values yield higher sparsity (more zeros) and smaller compressed sizes. Typical tested values are $1e-10$, $5e-11$, and $2.5e-11$ [S1].

**Steps:**

1.  **Level-wise Separation:**
    Decompose the mask according to the DWT hierarchy. Separate the mask into subbands corresponding to each DWT level. For the final (finest) DWT level, further separate the approximation subband (LL) from the detail subbands (LH, HL, HH, often called precincts) because they exhibit significantly different sparsity patterns (LL has fewer zeros) [S1].

2.  **8-bit Casting:**
    Pack the binarized mask values into groups of 8 bits. This step assumes that the majority of elements are zeros. By packing 8 bits into a single byte, long runs of zeros are represented more efficiently in the subsequent RLE step. For example, 1000 zeros are encoded as a single pair `(0, 125)` instead of three pairs `(0, 255), (0, 255), (0, 235)` [S1].

3.  **Scanning Order (Optional):**
    Flatten the 2D or 3D mask blocks into 1D streams using a scanning order. Options include zigzag, Morton (Z-order), spiral, or no scan. Note that the scanning order does not noticeably affect the final compressed size [S1].

4.  **Run-Length Encoding (RLE):**
    Apply RLE to the 8-bit casted stream. RLE encodes consecutive identical values as a pair `(value, count)`. This exploits the high sparsity (long runs of zeros) introduced by the learned masks [S1].

5.  **Huffman Encoding:**
    Apply Huffman coding to the RLE output stream. This maps high-probability values (such as the count for zero runs) to shorter bit lengths. Adaptive arithmetic coding was evaluated but discarded due to higher computational cost without meaningful size improvement [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_mask_pipeline(mask: List[List[int]], levels: int) -> bytes:
    """
    Compress a binarized learnable mask using the Mask Compression Pipeline.
    
    Args:
        mask: A list of lists (2D) or nested lists (3D) containing 0s and 1s.
        levels: The number of DWT decomposition levels.
        
    Returns:
        Compressed bytes.
    """
    compressed_chunks = []
    
    # Step 1: Level-wise separation
    # In a real implementation, 'mask' would be pre-decomposed into subbands
    # based on the DWT structure. Here we assume 'subbands' is a list of 
    # 2D arrays corresponding to each level and subband type.
    subbands = decompose_mask_into_subbands(mask, levels)
    
    for subband in subbands:
        # Step 2: 8-bit casting
        # Pack bits into bytes. Assuming most are 0.
        # This converts a list of 0/1 integers into a list of 8-bit integers.
        packed_bytes = pack_8bit(subband)
        
        # Step 3: Scanning order (Optional, negligible impact)
        # Flatten the 2D subband into 1D. 
        # Order can be zigzag, morton, spiral, or row-major.
        flat_stream = flatten_2d(packed_bytes, order='zigzag')
        
        # Step 4: Run-Length Encoding
        # Encode consecutive identical values as (value, count)
        rle_stream = run_length_encode(flat_stream)
        
        # Step 5: Huffman Encoding
        # Build Huffman tree based on frequency in rle_stream
        # and encode the stream.
        huffman_bits = huffman_encode(rle_stream)
        
        compressed_chunks.append(huffman_bits)
        
    return concatenate_chunks(compressed_chunks)

def pack_8bit(bit_array: List[int]) -> List[int]:
    """
    Pack a list of 0/1 integers into 8-bit integers.
    Example: [0,0,0,0,0,0,0,0] -> [0]
    """
    packed = []
    for i in range(0, len(bit_array), 8):
        chunk = bit_array[i:i+8]
        # Pad with zeros if less than 8 bits remain
        while len(chunk) < 8:
            chunk.append(0)
        # Convert 8 bits to integer
        byte_val = 0
        for bit in chunk:
            byte_val = (byte_val << 1) | bit
        packed.append(byte_val)
    return packed

def run_length_encode(stream: List[int]) -> List[Tuple[int, int]]:
    """
    Encode stream as list of (value, count) tuples.
    """
    if not stream:
        return []
    
    rle = []
    current_val = stream[0]
    count = 1
    
    for val in stream[1:]:
        if val == current_val:
            count += 1
        else:
            rle.append((current_val, count))
            current_val = val
            count = 1
    rle.append((current_val, count))
    return rle

def huffman_encode(rle_stream: List[Tuple[int, int]]) -> bytes:
    """
    Encode RLE stream using Huffman coding.
    """
    # Count frequencies
    freq = {}
    for val, count in rle_stream:
        freq[(val, count)] = freq.get((val, count), 0) + 1
        
    # Build Huffman Tree
    # (Implementation details omitted for brevity, standard algorithm)
    tree = build_huffman_tree(freq)
    
    # Encode
    bits = ""
    for val, count in rle_stream:
        bits += get_huffman_code(tree, (val, count))
        
    # Convert bits to bytes
    return bits_to_bytes(bits)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The 8-bit casting assumes most elements are zeros. If the mask is not sufficiently sparse, the compression ratio may degrade, though the method remains valid [S1].
*   **Boundary Conditions:** When packing the last group of bits in 8-bit casting, padding with zeros is required. The decoder must know the original bit length to discard padding [G].
*   **Subband Separation:** Treating LL and detail subbands separately is crucial because their sparsity differs significantly. Combining them would reduce the efficiency of the RLE step [S1].

## Subtopics

### Sparse Representation and Model Compression
A variant of the mask compression pipeline applied to direction-aware representations (e.g., DaReNeRF) using Dual-Tree Complex Wavelet Transforms (DTCWT) [S2]. It differs from the standard pipeline by applying learned masks to 12 direction-aware wavelet coefficients (6 real, 6 imaginary) plus 1 approximation coefficient, rather than standard DWT subbands [S2]. The sparsity is enforced via a straight-through estimator during training: $\tilde{W} = sg(H(M) - \text{sigmoid}(M)) \odot W$, where $M$ is the mask and $W$ is the coefficient [S2]. The compression steps (8-bit casting, RLE, Huffman) remain identical to the standard pipeline [S2]. This variant achieves significant model size reduction (e.g., 252 MB vs 1,210 MB for DaReNeRF) while maintaining rendering quality [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Compression Time:** Linear with respect to the number of mask elements. RLE and Huffman encoding are efficient operations. Huffman encoding requires a pass to build the frequency table, but adaptive arithmetic coding was discarded due to higher cost [S1].
*   **Decompression Time:** Linear. Inverse Huffman decoding and RLE decoding are fast.
*   **Memory:** The compressed mask size is significantly reduced. On the NeRF synthetic dataset, the pipeline achieves an average size of 0.28 MB, which is 7.39 times smaller than the original mask size [S1].

**Measured Performance:**
*   **RLE:** Reduces mask size by a factor of 1.75 on average [S1].
*   **Huffman:** Raises compression ratio to 4.62 [S1].
*   **8-bit Casting:** Increases ratio to 6.24 [S1].
*   **Level-wise Encoding:** Results in an average size of 0.28 MB (7.39x smaller than original) [S1].
*   **Scanning Order:** No noticeable effect on output size (e.g., 0.83 MB for no scan, zigzag, spiral vs 0.85 MB for Morton) [S1].

**Strengths and Limitations:**
*   **Strengths:** Extremely effective for highly sparse binary masks. Low computational overhead for both compression and decompression. Simple to implement.
*   **Limitations:** Relies on the mask being highly sparse (achieved via training regularization). Not suitable for dense data. The compression ratio is fixed by the Huffman table; dynamic updates are not supported in the basic pipeline [S1].

**When to Use:**
*   Use when storing learnable binary masks for neural fields (TensoRF, TriPlane, DaReNeRF) where sparsity is enforced during training.
*   Use when memory bandwidth or storage is a bottleneck, and the computational cost of compression/decompression must be low.

**When Not to Use:**
*   Do not use for dense volumetric data without sparsification.
*   Do not use if adaptive coding is required for non-stationary statistics (though Huffman is generally sufficient for masks) [S1].

## Background
The Mask Compression Pipeline was introduced in the context of compact neural radiance fields, specifically to address the storage overhead of learnable masks in methods like TensoRF and TriPlane [S1]. The method leverages the observation that learned masks are highly sparse and exhibit block-wise structures, making them ideal candidates for RLE and Huffman coding [S1]. The variant using DTCWT and direction-aware masks was later applied to dynamic scene representation in DaReNeRF [S2]. The pipeline builds on the shared principles of wavelet and pyramid transforms, specifically the use of hierarchical decomposition and entropy encoding [Parent].

## Sources
[S1] Masked Wavelet Representation for Compact Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.01981, fetched 2026-10-26
[S2] DaReNeRF: Direction-aware Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52733.2024.00481, fetched 2026-10-27
