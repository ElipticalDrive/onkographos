# Zerobit Encoding Scheme

Also called: Enhanced Encoding Scheme

## Overview
The Zerobit Encoding Scheme is a value compression method for volume rendering data that improves compression ratio and random access speed by exploiting spatial coherence in wavelet coefficients [S1, unofficial]. It modifies the cell tag table to indicate whether the coefficient vector $c_0$ is zero, reducing storage waste for zero-valued coefficients [S1, unofficial]. It sets itself apart from its siblings by using a variable-length significance map driven by "zerobits" (flags indicating if all 7 detail coefficients of a node are zero) rather than fixed-size maps or purely predictive differential coding [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 16x16x16 unit block of wavelet coefficients, previously decomposed into a set of 4x4x4 subblocks called "cells" [S1, unofficial].
*   **Output:** A compressed bitstream containing cell tags, cell information (zerobits and significance maps), and byte streams for coefficients [S1, unofficial].

**Data Structures:**
1.  **Cell Tag Table:** One byte per cell. The left-most bit (Bit 7) indicates the zerobit of the Level 0 node. The bit adjacent to the left-most bit (Bit 6) indicates the bytetag of $c_0$ for CT data, or whether $c_0$ is $(0,0,0)$ for RGB data. The remaining 6 bits (Bits 0-5) store the cell enumeration index (0 to 63) [S1, unofficial].
2.  **Cell Information:** Stored in one-byte blocks. The first byte contains 9 zerobits (1 for Level 0, 8 for Level 1 nodes). Subsequent bytes are significance maps. The number of significance map bytes equals the number of zerobits with value 1 (up to 9 bytes) [S1, unofficial].
3.  **Significance Map:** One byte per node where the zerobit is 1. Contains 7 bit-flags for the 7 detail coefficients of that node. The left-most bit (Bit 7) is a two-byte flag: 0 indicates at least one coefficient in that node is stored in a two-byte stream; 1 indicates all are one-byte [S1, unofficial].
4.  **Bytetag:** A single byte per cell. Tracks which specific detail coefficients (among those with significance flag 1) are stored in the two-byte stream. A value of 1 in the bytetag means the corresponding detail coefficient is stored in two-byte stream [S1, unofficial].
5.  **Byte Streams:** Two streams per cell (one-byte stream and two-byte stream). Each stream maintains an offset value in the cell information to locate coefficients [S1, unofficial].

**Numbered Steps (Encoding):**
1.  Enumerate cells front-to-back, top-to-bottom, left-to-right. Tag cells with 0 if all coefficients are zero, or with positive integers in increasing order if they contain non-zero coefficients [S1, unofficial].
2.  For each cell, determine the zerobit for each of the 9 nodes in the octree. If a node's zerobit is 0, all 7 detail coefficients are zero; no significance map is needed for that node [S1, unofficial].
3.  Store the 9 zerobits in the first byte of the Cell Information [S1, unofficial].
4.  For each node with zerobit 1, create a significance map byte. Set the 7 bits corresponding to the 7 detail coefficients. If any coefficient in that node is stored in a two-byte stream, set the left-most bit of this byte to 0 [S1, unofficial].
5.  Store the significance map bytes immediately after the zerobit byte. The total number of significance map bytes equals the number of zerobits equal to 1 [S1, unofficial].
6.  Determine the bytetag. For each coefficient with significance flag 1, if it requires two bytes, set the corresponding bit in the bytetag to 1 [S1, unofficial].
7.  Store coefficient $c_0$. For CT data, it is always stored in the one-byte stream (even if zero). For RGB data, $c_0$ is a 3-component vector $(r,g,b)$ stored in a three-byte stream, and the adjacent bit in the cell tag indicates if it is $(0,0,0)$ [S1, unofficial].
8.  Store detail coefficients. Coefficients with significance flag 1 are stored in either one-byte or two-byte streams based on the two-byte flag in the significance map and the bytetag [S1, unofficial].

**Decoding Algorithm:**
To decode a coefficient with index $(i, j, k)$ in a 16x16x16 unit block:
1.  Find the cell $C$ containing $(i, j, k)$.
2.  Check Cell Tag Table for $C$. If tag is 0, return 0 [Case 1] [S1, unofficial].
3.  Compute local index $(i', j', k')$ within cell $C$.
4.  Check the zerobit for $(i', j', k')$ in the Cell Information. If 0, return 0 [Case 2] [S1, unofficial].
5.  Check the bit-flag in the significance map for $(i', j', k')$. If 0, return 0 [Case 3] [S1, unofficial].
6.  If 1, the coefficient is non-zero. Calculate the address in the byte stream:
    *   Determine the displacement: the number of preceding non-zero coefficients (with flag 1) in the enumeration order.
    *   Use a precomputed indexing table $T(*)$ with 256 entries to count set bits in significance map bytes quickly [S1, unofficial].
    *   Address = Offset of the appropriate stream + Displacement [S1, unofficial].
    *   Return the value from the stream [Case 4] [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_coefficient(unit_block, i, j, k):
    """
    Decodes a wavelet coefficient from a Zerobit encoded unit block.
    unit_block: dict containing 'cell_tags', 'cell_info', 'stream_offsets', 'streams'
    i, j, k: global coordinates in the 16x16x16 block
    """
    # 1. Find the cell C
    cell_x, cell_y, cell_z = i // 4, j // 4, k // 4
    cell_idx = cell_z * 16 + cell_y * 4 + cell_x
    
    # 2. Check Cell Tag
    cell_tag = unit_block['cell_tags'][cell_idx]
    if cell_tag == 0:
        return 0  # Case 1
    
    # 3. Get Cell Information
    cell_info = unit_block['cell_info'][cell_idx]
    zerobits_byte = cell_info[0]
    
    # Local coordinates in cell
    local_i, local_j, local_k = i % 4, j % 4, k % 4
    
    # c0 is at (0,0,0)
    if local_i == 0 and local_j == 0 and local_k == 0:
        # Check zero flag for c0 in cell tag (Bit 6, assuming Bit 7 is Level 0 zerobit)
        is_zero_c0 = (cell_tag >> 6) & 1
        if is_zero_c0:
            return 0
        # Retrieve from stream. c0 is always first in one-byte stream for CT, or 3-byte for RGB.
        offset = unit_block['stream_offsets'][cell_idx]['one_byte']
        return unit_block['streams']['one_byte'][offset]

    # Map local coordinates to Level 1 node index (1-8) and detail index (0-6)
    # Node index corresponds to the 2x2x2 subblock position
    node_idx = 4 * local_k + 2 * local_j + local_i + 1
    
    # Check zerobit for this node (Bits 1-8 of zerobits_byte)
    zerobit = (zerobits_byte >> node_idx) & 1
    if zerobit == 0:
        return 0  # Case 2
    
    # Find the significance map byte for this node
    sig_map_offset = 1  # Start after zerobit byte
    for n in range(1, 9):
        zb = (zerobits_byte >> n) & 1
        if n == node_idx:
            break
        if zb == 1:
            sig_map_offset += 1
            
    sig_map_byte = cell_info[sig_map_offset]
    
    # Determine the detail coefficient index within the node [G]
    # Standard wavelet ordering maps the 7 non-average positions in a 2x2x2 block to indices 0-6.
    detail_idx = (local_k * 4 + local_j * 2 + local_i) % 7
    
    # Check the bit in sig_map_byte
    flag = (sig_map_byte >> detail_idx) & 1
    if flag == 0:
        return 0  # Case 3
        
    # Case 4: Retrieve value
    # Calculate displacement: number of preceding non-zero coefficients with flag 1
    displacement = 0
    # Count set bits in preceding significance maps
    for n in range(1, node_idx):
        zb = (zerobits_byte >> n) & 1
        if zb == 1:
            # Find sig map for node n
            temp_offset = 1
            for m in range(1, n):
                if (zerobits_byte >> m) & 1:
                    temp_offset += 1
            temp_sig = cell_info[temp_offset]
            # Count set bits in this sig map byte using T(*) table
            displacement += T[temp_sig]
            
    # Add set bits in current sig map up to detail_idx
    mask = (1 << detail_idx) - 1
    displacement += T[sig_map_byte & mask]
    
    # Determine stream (one-byte or two-byte)
    two_byte_flag = (sig_map_byte >> 7) & 1
    if two_byte_flag == 0:
        bytetag = cell_info[sig_map_offset + 1]
        is_two_byte = (bytetag >> detail_idx) & 1
        stream = 'two_byte' if is_two_byte else 'one_byte'
    else:
        stream = 'two_byte'
        
    offset = unit_block['stream_offsets'][cell_idx][stream]
    address = offset + displacement
    
    return unit_block['streams'][stream][address]

# Precomputed popcount table T(*) with 256 entries [S1, unofficial]
T = [bin(i).count('1') for i in range(256)]
```

**Parameters:**
*   **Indexing Table T(*):** A precomputed table with 256 entries. Given a one-byte word, it returns the number of bit 1s in the byte [S1, unofficial].
*   **Zerobit Ratio:**
    *   CT Data (3.0% desired ratio): Level 0 zerobit ratio 0.230, Level 1 zerobit ratio 0.949 [S1, unofficial].
    *   RGB Data (1.0% desired ratio): Level 0 zerobit ratio 0.754, Level 1 zerobit ratio 0.993 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **CT Data $c_0$:** Stored in the one-byte stream even if zero, causing unnecessary storage overhead [S1, unofficial].
*   **RGB Data:** $c_0$ range [0,255] requires 3-byte streams (unsigned char). Detail vectors range [-128,127] require 3-byte streams (signed char) [S1, unofficial].
*   **Precision:** The scheme relies on wavelet coefficients being thresholded to zero beforehand [S1, unofficial].

## Operating characteristics
*   **Compression Ratio:** Improves over previous schemes using fixed 8-byte significance maps by using variable-length maps (average < 8 bytes for non-null cells) [S1, unofficial].
*   **Random Access Speed:** Supports fast random access by allowing early termination (Cases 1-3) during decoding without reading the full byte stream [S1, unofficial].
*   **Memory:** Requires storage for Cell Tag Table (1 byte/cell), Cell Information (variable), and Byte Streams [S1, unofficial].
*   **Strengths:** High compression ratio, fast random access, no image quality loss [S1, unofficial].
*   **Limitations:** Complexity in decoding due to variable-length maps and bytetag handling [S1, unofficial].

## Background
The Zerobit Encoding Scheme (also called Enhanced Encoding Scheme) was proposed to improve upon previous wavelet-based volume compression methods [S1, unofficial]. It leverages the spatial coherence of wavelet coefficients, where null coefficients often form thick clusters [S1, unofficial]. It is particularly effective for interactive visualization systems [S1, unofficial].

## Sources
[S1] Compression Scheme, https://www.nlm.nih.gov/research/visible/vhpconf98/AUTHORS/PARK/SECTION3.HTM, fetched 2026-10-28 (unofficial)
