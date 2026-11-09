# Hierarchical Block Significance Map with Offset Arrays

Also called: Block 32/8 Significance Map, Slice Info / Block 32 Info / Block 8 Info structure

## Overview
This method provides fast random access to wavelet coefficients stored in compressed linear streams, overcoming the variable-length decoding overhead inherent in Huffman or arithmetic coding [S1]. It achieves this by organizing significant coefficients into a hierarchy of quadratic blocks (32x32, then 8x8) and lines, using bitmaps and split offset arrays to locate data directly [S1]. It sets itself apart from sibling methods by explicitly partitioning the offset space across three record levels to minimize storage overhead while maintaining constant-time lookup probabilities [S1].

## How it works

### Inputs and Outputs
*   **Input:** A 3D volume represented as wavelet coefficients, preprocessed with a specific wavelet transform (e.g., Haar) and quantized.
*   **Output:** A set of four linear streams (Block 32 Stream, Block 8 Stream, Significance Map Stream, Byte Stream) and three types of metadata records (Slice Info, Block 32 Info, Block 8 Info) that allow retrieving any coefficient $(x, y, z)$ without decoding preceding data.

### Data Structures and Layout

The structure is hierarchical. The volume is divided into slices $z$. Each slice is divided into $32 \times 32$ blocks. Each block is divided into $8 \times 8$ sub-blocks.

**1. Slice Info Record**
Stored in an array indexed by slice $z$.
*   **Block 32 Significance Map:** A fixed-size bitmap for the slice. Each bit corresponds to a $32 \times 32$ block. If the bit is 1, the block contains nonzero coefficients; if 0, the block is all zeros.
*   **Block 32 Offset:** A 16-bit integer pointing to the start of the corresponding Block 32 Info record in the Block 32 Stream.
*   **Line Offset Array:** An array of integers. The entry for line $y$ contains the count of nonzero $32 \times 32$ blocks preceding line $y$ in the Block 32 Significance Map. This allows calculating the position of a block within the stream based on its line.
*   **Significance Map Offset (High Bits):** The most significant bits of the 32-bit offset into the Significance Map Stream. The exact split is defined below.
*   **MSB Flag:** The Most Significant Bit (MSB) of the Block 32 Offset indicates the direction of prediction for I-slices [S1].

**2. Block 32 Info Record**
Stored in the Block 32 Stream. One record exists for each nonzero $32 \times 32$ block.
*   **Block 8 Significance Map:** A bitmap for the $32 \times 32$ block. Each bit corresponds to an $8 \times 8$ sub-block.
*   **Block 8 Offset:** A 16-bit integer pointing to the start of the corresponding Block 8 Info record in the Block 8 Stream.
*   **Significance Map Offset (Middle Bits):** 8 bits of the 32-bit offset into the Significance Map Stream.
*   **Byte Stream Offset (Low Bits):** The least significant 16 bits of the offset into the Byte Stream. Note: This is insufficient for the full offset (see Pitfalls).

**3. Block 8 Info Record**
Stored in the Block 8 Stream. One record exists for each nonzero $8 \times 8$ sub-block.
*   **Zeroline Map:** A bitmap of size 8 bits (one per line in the $8 \times 8$ sub-block). If bit $j$ is 0, line $j$ contains at least one nonzero coefficient. If bit $j$ is 1, line $j$ is all zeros.
*   **Significance Map Offset (Low Bits):** 16 bits of the 32-bit offset into the Significance Map Stream.
*   **Byte Stream Offset (MSB):** The 2 Most Significant Bits of the full Byte Stream Offset are stored here. The remaining 16 bits are in the Block 32 Info record.
*   **Encoding Mode Flag:** The MSB of the Byte Stream Offset (which is now the 17th bit conceptually, but stored in the MSB position of the 18-bit combined offset) indicates the encoding mode:
    *   `1`: Coefficients are stored using the displacement method (offset $\theta$ + signed displacements).
    *   `0`: Coefficients are stored using the quantization method (A, B, x).

**4. Streams**
*   **Block 32 Stream:** Concatenation of all Block 32 Info records.
*   **Block 8 Stream:** Concatenation of all Block 8 Info records.
*   **Significance Map Stream:** Concatenation of 1-byte significance maps. For each nonzero line in a nonzero sub-block, 1 byte is stored per coefficient in that line (8 bytes per line).
*   **Byte Stream:** Concatenation of coefficient data.

### Parameters
*   **Block Size ($B$):** 32. Chosen because Table 3 in [S1] shows that for PSNR 46, 38.4% of blocks are nonzero, balancing sparsity and overhead.
*   **Sub-block Size:** 8x8. Chosen because Table 4 in [S1] shows a 67.6% probability (at PSNR 46) that an 8x8 sub-block is nonzero given its 32x32 parent is nonzero, justifying the second level of hierarchy.
*   **R-slice Spacing ($F$):** Optimal spacing is $F=4$ ($k=2$, where $F=2^k$). This yields the smallest compressed size (6.01 Mb at PSNR 46) compared to other spacings [S1].

### Numbered Steps for Random Access

To retrieve the wavelet coefficient at coordinates $(x, y, z)$:

1.  **Lookup Slice Info:** Retrieve the `Slice Info` record for slice $z$.
2.  **Check Block 32 Significance:**
    *   Calculate the block index $(bx, by)$ in the $32 \times 32$ grid: $bx = \lfloor x/32 \rfloor, by = \lfloor y/32 \rfloor$.
    *   Check the bit at position $(by \times \text{width} + bx)$ in the `Block 32 Significance Map`.
    *   If the bit is 0, the coefficient is 0. Return 0.
3.  **Locate Block 32 Info:**
    *   Calculate the number of set bits preceding the current block in the same bitmap line. This is done by summing the `Line Offset` entries for lines $0$ to $by-1$, plus the count of set bits in the current line from column 0 to $bx-1$.
    *   Use a precomputed bit-count table (size 65,536 entries, $2^{16}$) to count bits in the current line segment efficiently.
    *   Add this count to the `Block 32 Offset` from the `Slice Info` record.
    *   Use this sum as the index into the `Block 32 Stream` to retrieve the `Block 32 Info` record.
4.  **Check Block 8 Significance:**
    *   Calculate the sub-block index $(sbx, sby)$ within the $32 \times 32$ block: $sbx = \lfloor (x \pmod{32})/8 \rfloor, sby = \lfloor (y \pmod{32})/8 \rfloor$.
    *   Check the bit at position $(sby \times 4 + sbx)$ in the `Block 8 Significance Map` of the retrieved record.
    *   If the bit is 0, the coefficient is 0. Return 0.
5.  **Locate Block 8 Info:**
    *   Calculate the number of set bits preceding the current sub-block in the `Block 8 Significance Map`.
    *   Add this count to the `Block 8 Offset` from the `Block 32 Info` record.
    *   Use this sum as the index into the `Block 8 Stream` to retrieve the `Block 8 Info` record.
6.  **Check Zeroline Map:**
    *   Calculate the line index $ly$ within the $8 \times 8$ sub-block: $ly = (y \pmod 8)$.
    *   Check the bit at position $ly$ in the `Zeroline Map`.
    *   If the bit is 1 (line is zero), the coefficient is 0. Return 0.
7.  **Locate Significance Map:**
    *   Construct the full 32-bit `Significance Map Offset` by concatenating:
        *   High bits from `Slice Info`.
        *   Middle 8 bits from `Block 32 Info`.
        *   Low 16 bits from `Block 8 Info`.
    *   The position in the `Significance Map Stream` is determined by this offset plus the offset of the specific line within the sub-block.
    *   Retrieve the 1-byte significance map for the line.
8.  **Check Coefficient Significance:**
    *   Calculate the coefficient index $lx$ within the line: $lx = (x \pmod 8)$.
    *   Check the bit at position $lx$ in the retrieved significance map byte.
    *   If the bit is 0, the coefficient is 0. Return 0.
9.  **Retrieve Coefficient Value:**
    *   Construct the full Byte Stream Offset by concatenating:
        *   Low 16 bits from `Block 32 Info`.
        *   High 2 bits from `Block 8 Info` (stored in the MSB position of the offset field).
    *   Access the `Byte Stream` at this offset.
    *   Decode the value based on the encoding mode flag (MSB of the combined offset):
        *   If mode is 1: Read 2-byte offset $\theta$. Read 1-byte signed displacement $d$ for the coefficient. Value = $\theta + d$ (or $-\theta - d$ depending on sign convention defined in [S1], specifically intervals $[\theta, \theta+127]$ or $[-\theta-128, -\theta]$).
        *   If mode is 0: Read 2-byte $A$, 2-byte $B$, 1-byte $x$. Decode using the quantization formula from the parent method [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_wavelet_coefficient(x, y, z):
    # 1. Lookup Slice Info
    slice_info = lookup_slice_info(z)
    
    # 2. Check Block 32 Significance
    bx = x // 32
    by = y // 32
    
    # Check if block is zero
    if not is_block_nonzero(bx, by, slice_info.block_32_significance_map):
        return 0
        
    # 3. Locate Block 32 Info
    # Calculate index in Block 32 Stream
    # Sum Line Offsets for previous lines
    line_offset_sum = sum(slice_info.line_offset_array[i] for i in range(by))
    
    # Count set bits in current line before bx
    # Assuming bitmap is stored as array of 16-bit words for table lookup
    current_line_bits = get_bits_in_range(slice_info.block_32_significance_map, by, 0, bx)
    
    block_32_index = line_offset_sum + current_line_bits + slice_info.block_32_offset
    block_32_info = lookup_block_32_stream(block_32_index)
    
    # 4. Check Block 8 Significance
    sbx = (x % 32) // 8
    sby = (y % 32) // 8
    
    if not is_block_nonzero(sbx, sby, block_32_info.block_8_significance_map):
        return 0
        
    # 5. Locate Block 8 Info
    # Count set bits in Block 8 map before current sub-block
    sub_block_index = sby * 4 + sbx
    # Count bits in 4x4 grid (16 bits)
    bits_before = count_bits_in_16_bit_word(block_32_info.block_8_significance_map, sub_block_index)
    
    block_8_index = bits_before + block_32_info.block_8_offset
    block_8_info = lookup_block_8_stream(block_8_index)
    
    # 6. Check Zeroline Map
    ly = y % 8
    if is_zeroline(ly, block_8_info.zeroline_map):
        return 0
        
    # 7. Locate Significance Map
    # Construct 32-bit Significance Map Offset
    sig_map_offset = (slice_info.sig_map_offset_high << 24) | \
                     (block_32_info.sig_map_offset_mid << 16) | \
                     (block_8_info.sig_map_offset_low)
                     
    # Add offset for the specific line within the sub-block
    # Each line has 8 coefficients, so 8 bytes in Sig Map Stream
    sig_map_pos = sig_map_offset + (ly * 8)
    sig_map_byte = lookup_sig_map_stream(sig_map_pos)
    
    # 8. Check Coefficient Significance
    lx = x % 8
    if not is_coefficient_significant(lx, sig_map_byte):
        return 0
        
    # 9. Retrieve Coefficient Value
    # Construct Byte Stream Offset
    # Low 16 bits from Block 32 Info
    # High 2 bits from Block 8 Info (MSB of its offset field)
    byte_offset_low = block_32_info.byte_stream_offset
    byte_offset_high = block_8_info.byte_stream_offset_msb # 2 bits
    
    full_byte_offset = (byte_offset_high << 16) | byte_offset_low
    
    # Determine encoding mode from the MSB of the combined 18-bit offset
    # The MSB of the 18-bit offset is bit 17.
    # In the structure, this is stored in the MSB of the Block 8 Offset field.
    encoding_mode = (block_8_info.byte_stream_offset_msb >> 1) & 1
    
    if encoding_mode == 1:
        # Displacement encoding
        # Read 2-byte offset theta
        theta = read_16bit_signed(byte_stream, full_byte_offset)
        # Read 1-byte signed displacement
        displacement = read_8bit_signed(byte_stream, full_byte_offset + 2)
        # Determine sign based on interval logic
        # If in [theta, theta+127], value = theta + displacement
        # If in [-theta-128, -theta], value = -theta - displacement
        # Note: Exact sign logic depends on specific implementation of theta sign bit
        # Assuming theta is positive magnitude and sign is derived from context or additional bit
        # Simplified: value = theta + displacement (adjust as per [S1] specific interval check)
        value = theta + displacement
    else:
        # Quantization encoding
        A = read_16bit(byte_stream, full_byte_offset)
        B = read_16bit(byte_stream, full_byte_offset + 2)
        x_val = read_8bit(byte_stream, full_byte_offset + 4)
        value = decode_quantized(A, B, x_val)
        
    return value

def count_bits_in_16_bit_word(bitmap, index):
    # Helper to count set bits in a 16-bit segment of the bitmap
    # Uses precomputed table of 65536 entries
    word = extract_16_bit_word(bitmap, index)
    return bit_count_table[word]
```

### Edge Cases and Pitfalls
*   **Byte Stream Offset Overflow:** 18 bits are required to offset all coefficients in a slice, but only 16 bits are available in the Block 32 Info record. The solution is to store the last 2 MSBs of the Byte Stream Offset in the MSB of the Block 8 Offset field [S1].
*   **Precision:** Coefficients are scaled and rounded to integer intervals $[-4095, 4095]$ before encoding [S1].
*   **Boundary Conditions:** The bit-counting logic must correctly handle the boundaries of the bitmap lines and the 16-bit word alignment for the precomputed table [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Lookup Time:** On average, approximately 2 lookups in encoded data and 1 lookup in the bitcount table are performed to retrieve a single coefficient [S1]. The empirical probability of reaching each stage is roughly halved at each step (100%, 50%, 25%, 12.5%, 6%) [S1].
*   **Memory:** The overhead of the offset arrays is minimized by splitting offsets across the hierarchy. Block 32 Offset is 16 bits (in Block 32 Info) + 32 bits (in Slice Info) for Significance Map. Block 8 Offset is 16 bits (in Block 32 Info). Significance Map Offset is split 8 bits (Block 32 Info) + 16 bits (Block 8 Info) + remaining bits (Slice Info) [S1].

### Measured Performance
*   **Hardware/Scene:** Visible Human dataset [S1].
*   **Compression Factors:**
    *   PSNR 46: Factor 42.6 (Size 6.01 Mb) [S1].
    *   PSNR 43: Factor 60.2 (Size 4.26 Mb) [S1].
    *   PSNR 55.8: Factor 14.5 (Size 17.68 Mb) [S1].
*   **Reconstruction Cost:** Decoding a voxel in an I-slice requires retrieving 14 wavelet coefficients (7 from I-slice, 7 from R-slice) and performing 12 additions on average [S1].

### Strengths and Limitations
*   **Strengths:** Fast random access without variable-length decoding overhead. Significant size reduction (approx. 50% compared to [12, 11]) by setting more coefficients to zero for a given PSNR [S1].
*   **Limitations:** Complexity of the data structure and offset management. The method is tailored for CT images and may require adjustment for other volume types [S1].

## Background

### Origin
The method was proposed in the paper "Wavelet Based 3D Compression for Very Large Volume Data Supporting Fast Random Access" [S1]. It builds on previous work [12, 11] which used Huffman or arithmetic coding but lacked fast random access capabilities [S1].

### History and Development
The method addresses the limitation of variable-length codes (Huffman, arithmetic, run-length) which do not lend themselves to fast retrieval of individual coefficients [S1]. It was designed for preprocessed CT images of the Visible Human dataset but is extensible to other volumes [S1].

## Sources

[S1] Wavelet Based 3D Compression for Very Large Volume Data Supporting Fast Random Access, https://www.brics.dk/RS/99/34/index.html, fetched 2026-11-04
