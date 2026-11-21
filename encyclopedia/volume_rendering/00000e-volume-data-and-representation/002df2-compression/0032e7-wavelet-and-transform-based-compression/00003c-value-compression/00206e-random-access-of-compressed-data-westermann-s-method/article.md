# Random Access of Compressed Data (Westermann's Method)
Also called: Mask and Offset Layout for Wavelets, Westermann's Method

## Overview
This method provides a data layout technique to support random access to wavelet coefficients stored in compressed linear arrays, overcoming the variable-length decoding overhead inherent in Huffman or arithmetic coding [S1, unofficial]. It achieves this by storing entries as linear arrays per level, each containing a leading mask and quantized coefficients, supplemented by offset arrays to indicate byte distances between streams [S1, unofficial]. It sets itself apart from sibling methods by explicitly using masks and offsets for direct traversal, introducing memory overhead to minimize sequential search requirements [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume $V$ decomposed into wavelet coefficients $C$ across multiple resolution levels [S1, unofficial].
*   **Output:** A compressed linear byte stream supporting $O(1)$ random access to any coefficient, along with auxiliary mask and offset structures [S1, unofficial].

**Data Structures:**
The data is stored as linear arrays per level [S1, unofficial]. Each entry in the array corresponds to a spatial location and contains:
1.  **Mask:** A leading bit-field defining which of the up to seven coefficients per entry were removed or are zero [S1, unofficial].
2.  **Coefficients:** The quantized wavelet coefficients themselves [S1, unofficial].
3.  **Offsets:** Additional arrays (`OFFx`, `OFFy`, `OFFz`) indicating the byte distance between streams or blocks to enable direct indexing [S1, unofficial].

**Steps:**
1.  **Quantization:** Quantize the wavelet coefficients, which are concentrated around zero, into 2 or 3 bits using non-uniform quantization [S1, unofficial].
2.  **Packing:** Pack the quantized values into bytes; for 2-bit quantization, four coefficients fit into one byte, achieving a 4:1 compression ratio for the coefficient data itself [S1, unofficial].
3.  **Masking:** For each entry, generate a leading mask that defines which coefficients have been removed or are zero [S1, unofficial].
4.  **Run-Length Coding:** Apply run-length coding to entries that consist entirely of zero coefficients to further compress the stream [S1, unofficial].
5.  **Offset Calculation:** Compute offset information indicating the number of bytes from a certain stream to another, allowing the renderer to jump directly to the required data block without sequential search [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def westermann_layout(volume_levels):
    # volume_levels: list of 3D arrays, each half resolution of previous
    # Each entry holds up to 7 coefficients
    streams, masks, offsets = [], [], []
    current_offset = 0
    
    for level in volume_levels:
        level_stream, level_masks = [], []
        for entry in level.flatten():
            # Quantize to 2 bits (non-uniform, concentrated around zero)
            quantized = [max(0, min(3, int(c * 2))) for c in entry]
            # Pack 4 coefficients per byte
            packed = [(quantized[i] << (2 * (i % 4))) & 0xFF for i in range(0, len(quantized), 4)]
            # Mask for removed/zero coefficients
            mask = sum(1 << i for i, c in enumerate(entry) if c != 0)
            # RLE for zero-only entries
            if all(c == 0 for c in entry):
                packed = [0] * len(packed)
            level_stream.append(packed)
            level_masks.append(mask)
        streams.append(level_stream)
        masks.append(level_masks)
        offsets.append(current_offset)
        current_offset += len(level_stream)
    return streams, masks, offsets
```

**Edge Cases and Pitfalls:**
*   **Precision:** Non-uniform quantization may introduce visible artifacts if the bit-depth is too low for the dynamic range of the data [S1, unofficial].
*   **Boundaries:** The mask must correctly handle boundary conditions where fewer than the maximum number of coefficients exist per entry [S1, unofficial].
*   **Degenerate Input:** If the volume contains no significant coefficients, the run-length coding should handle large blocks of zeros efficiently [S1, unofficial].

## Subtopics

### [Hierarchical Block Significance Map with Offset Arrays](8731-hierarchical-block-significance-map-with-offset-arrays.md)
This method provides fast random access to wavelet coefficients stored in compressed linear streams, overcoming the variable-length decoding overhead inherent in Huffman or arithmetic coding. It achieves this by organizing significant coefficients into a hierarchy of quadratic blocks (32x32, then 8x8) and lines, using bitmaps and split offset arrays to locate data directly. It sets itself apart from sibling methods by explicitly partitioning the offset space across three record levels to minimize storage overhead while maintaining constant-time lookup probabilities.

## Operating characteristics
*   **Time Complexity:** Random access is $O(1)$ after offset computation, as it avoids sequential decoding of variable-length codes [S1, unofficial].
*   **Memory Complexity:** Introduces overhead due to masks and offset arrays [S1, unofficial].
*   **Performance:** Achieves a 4:1 compression ratio with 2-bit quantization [S1, unofficial].
*   **Strengths:** Enables direct rendering without full decompression; efficient bit-wise extraction in C [S1, unofficial].
*   **Limitations:** Memory overhead from masks and offsets; potential quality loss from quantization [S1, unofficial].

## Background
Originated by Westermann [51] as a solution to the problem of random access in compressed wavelet data [S1, unofficial]. The method was proposed to overcome the limitations of Huffman coding, which requires sequential decoding, and run-length coding, which requires sequential search [S1, unofficial]. It is used in direct rendering of multiresolution decomposed volume data [S1, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
