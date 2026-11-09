# Zerobit Encoding Scheme

Also called: Enhanced Encoding Scheme

## Overview
The Zerobit Encoding Scheme is a compression method for wavelet-transformed volume data that improves compression ratio and random access speed by exploiting spatial coherence in wavelet coefficients [S1, unofficial]. It modifies the cell tag table to indicate whether the coefficient vector $c_0$ is zero, reducing storage waste for zero-valued coefficients [S1, unofficial]. Unlike previous schemes that used a fixed 8-byte significance map for all non-null cells, this scheme uses a variable-length significance map (up to 9 bytes) but averages less than 8 bytes for non-null cells, improving compression ratio [S1, unofficial]. Compared to previous methods, this scheme supports fast random access and high compression ratio without image quality loss [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 16x16x16 encoded unit block, which is a set of reformed 4x4x4 subblocks called cells. The input data is typically CT (scalar) or RGB (vector) wavelet coefficients.
*   **Output:** A compressed bitstream containing cell tags, cell information (zerobits and significance maps), and byte streams for coefficients.

**Data Structures:**
1.  **Cell Tag Table:** One byte per cell.
    *   Bit 0 (left-most): Zerobit of the node on level 0.
    *   Bit 1: Adjacent to Bit 0. For CT data, this is the bytetag of $c_0$. For RGB data, this indicates if $c_0$ vector is $(0,0,0)$.
    *   Bits 2-7: Unused or reserved in the tag byte itself, but the zerobits for level 1 nodes are stored in the Cell Information block.
2.  **Cell Information:** Stored in one-byte blocks.
    *   First byte: Contains 9 zerobits. Bit 0 is the level 0 zerobit (redundant with Cell Tag Table Bit 0 but stored here for alignment). Bits 1-8 are the zerobits for the 8 nodes on level 1.
    *   Subsequent bytes: Significance maps. The number of bytes equals the count of zerobits with value 1 (up to 8 bytes for level 1 nodes, plus potentially 1 for level 0 if its zerobit is 1).
3.  **Significance Map:** A variable-length map. Each byte corresponds to a node in the cell's octree that has a zerobit of 1.
    *   Bits 1-7: Flags for the 7 detail coefficients of that node. 1 means the coefficient is non-zero and stored; 0 means it is zero.
    *   Bit 0 (left-most): Two-byte flag. 0 indicates existence of at least one coefficient in that node stored in a two-byte stream. 1 indicates all coefficients in that node are stored in one-byte streams.
4.  **Bytetag:** A byte used to track which specific detail coefficients (among those with significance map flag 1) are stored in the two-byte stream. A value of 1 in the bytetag means the corresponding detail coefficient is stored in two-byte stream.
5.  **Byte Streams:** Two streams per cell (one-byte stream and two-byte stream). Each stream maintains an offset value in the cell information to locate coefficients.
6.  **Indexing Table T(*):** A precomputed table with $2^8 = 256$ entries. Given a one-byte word (8 bit flags), it returns the number of bit 1s in the byte.

**Numbered Steps (Encoding):**
1.  Enumerate cells front-to-back, top-to-bottom, left-to-right.
2.  Tag cells with zero if all coefficients are zero, or with positive integers if they contain non-zero coefficients.
3.  For each cell, determine the zerobit for each of the 9 nodes in the octree.
    *   If a node's zerobit is 0, all 7 detail coefficients are zero; no significance map is needed for that node.
    *   If a node's zerobit is 1, a significance map byte is required.
4.  Store the significance map only for nodes with zerobit 1. The number of bytes in the significance map equals the number of zerobits with value 1 (up to 9 bytes).
5.  For each byte in the significance map, use the left-most bit (Bit 0) to flag if any coefficient in that node is stored in two bytes. If so, update the bytetag to mark which specific coefficients are two-byte.
6.  Store coefficient $c_0$.
    *   For CT data, it is always stored in the one-byte stream (even if zero). The bytetag bit in the Cell Tag Table indicates its storage type.
    *   For RGB data, $c_0$ is a 3-component vector $(r,g,b)$ stored in a three-byte stream, and the adjacent bit in the cell tag indicates if it is $(0,0,0)$.
7.  Store detail coefficients. Coefficients with significance flag 1 are stored in either one-byte or two-byte streams based on the two-byte flag in the significance map and the bytetag.

**Decoding Algorithm:**
To decode a coefficient with index $(i, j, k)$ from a 16x16x16 encoded unit block:

1.  Find the cell $C$ containing the coefficient $(i, j, k)$.
2.  If its cell tag is 0, return 0. [Case 1]
3.  Compute the index $(i', j', k')$ in $C$.
4.  If the zerobit for $(i', j', k')$ is 0, return 0. [Case 2]
5.  If the bit-flag in significance map for $(i', j', k')$ is 0, return 0. [Case 3]
6.  Count the number of preceding non-zero coefficients by table access.
    *   Calculate displacement: sum of bits set to 1 in significance map bytes preceding the target coefficient's node, plus bits set to 1 in the target node's significance map preceding the target coefficient.
    *   Use the precomputed indexing table $T(*)$ to count set bits in each byte efficiently.
7.  Access the appropriate data stream (one-byte or two-byte, determined by the two-byte flag and bytetag), add the displacement to the stream's offset value, and return the value. [Case 4]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_coefficient(unit_block, i, j, k):
    """
    Decodes a wavelet coefficient at index (i, j, k) from a 16x16x16 unit block.
    Returns the coefficient value.
    """
    # 1. Find the cell C containing the coefficient (i, j, k)
    # A 16x16x16 block is composed of 4x4x4 = 64 cells of size 4x4x4.
    # Cell indices (cx, cy, cz) are derived from (i, j, k)
    cx = i // 4
    cy = j // 4
    cz = k // 4
    
    # Enumerate cells front-to-back, top-to-bottom, left-to-right
    # Assuming standard row-major order for cells:
    cell_index = cz * 16 + cy * 4 + cx
    
    # Retrieve cell tag from Cell Tag Table
    cell_tag = unit_block.cell_tag_table[cell_index]
    
    # Case 1: Cell is null
    if cell_tag == 0:
        return 0
    
    # 2. Compute local index (i', j', k') in C
    i_prime = i % 4
    j_prime = j % 4
    k_prime = k % 4
    
    # Retrieve Cell Information for this cell
    # The cell info block index is stored in the cell tag or a separate structure
    # Assuming unit_block.cell_info_blocks[cell_index] gives the block
    cell_info = unit_block.cell_info_blocks[cell_index]
    
    # Extract zerobits
    # First byte of cell_info contains 9 zerobits
    # Bit 0: Level 0 zerobit
    # Bits 1-8: Level 1 zerobits
    zerobits_byte = cell_info[0]
    
    # Map (i', j', k') to a specific node in the octree
    # The octree has 1 root (Level 0) and 8 children (Level 1).
    # Level 0 corresponds to the whole cell average c0.
    # Level 1 nodes correspond to 2x2x2 subblocks.
    # Determine if (i', j', k') falls into Level 0 or Level 1.
    # In the context of the paper, c0 is Level 0. The 7 detail coefficients
    # are associated with Level 1 nodes.
    # However, the decoding algorithm checks zerobit for the coefficient's index.
    # Let's assume the index maps to a specific node.
    # If (i', j', k') is the center/average, it's c0.
    # Otherwise, it belongs to one of the 8 Level 1 nodes.
    
    # Determine node index
    # Level 0 node index: 0
    # Level 1 node indices: 1 to 8
    # Mapping (i', j', k') to Level 1 node:
    # Each Level 1 node is a 2x2x2 block.
    # Node index = (i' // 2) * 4 + (j' // 2) * 2 + (k' // 2) + 1
    node_index = (i_prime // 2) * 4 + (j_prime // 2) * 2 + (k_prime // 2) + 1
    
    # Check Level 0 zerobit (Bit 0 of zerobits_byte)
    level_0_zerobit = (zerobits_byte >> 0) & 1
    
    # Check Level 1 zerobit (Bit corresponding to node_index)
    # node_index is 1-8. Map to bits 1-8.
    level_1_zerobit_bit = node_index # 1 to 8
    level_1_zerobit = (zerobits_byte >> level_1_zerobit_bit) & 1
    
    # Case 2: Zerobit is 0
    if node_index == 0:
        if level_0_zerobit == 0:
            return 0
    else:
        if level_1_zerobit == 0:
            return 0
            
    # 3. Check Significance Map
    # Significance maps follow the zerobits byte.
    # We need to find the significance map byte for the node.
    # The significance maps are stored for nodes with zerobit 1.
    # We need to count how many zerobits before this node were 1.
    
    # Reconstruct the list of zerobits for all 9 nodes
    zerobits = []
    zerobits.append(level_0_zerobit)
    for b in range(1, 9):
        zerobits.append((zerobits_byte >> b) & 1)
        
    # Find the index of the significance map byte for this node
    # The significance maps are stored sequentially for nodes with zerobit 1
    sig_map_index = -1
    count = 0
    for n in range(9):
        if zerobits[n] == 1:
            if n == node_index:
                sig_map_index = count
                break
            count += 1
            
    # If node_index is 0, it might not have a significance map byte in the same way
    # The paper says c0 is stored in one-byte stream.
    # For detail coefficients, we use the significance map.
    
    if node_index == 0:
        # c0 handling
        # Check if c0 is zero based on CT/RGB specific logic
        # For CT, c0 is always in one-byte stream.
        # For RGB, check the bit in cell tag.
        # Assuming CT for now, c0 is retrieved from stream.
        # But if zerobit is 1, it means c0 might be non-zero?
        # Actually, the zerobit for Level 0 is for the 7 detail coefficients of the root?
        # No, the paper says "c0 at root node... d0i on level 0 are detail coefficients".
        # Wait, the paper says "c0 at root node... d0i (i=1..7) on level 0".
        # And "c1j on level 1".
        # The zerobit is for "nodes".
        # Level 0 node has 7 detail coefficients.
        # Level 1 nodes have 7 detail coefficients each?
        # The paper says "7 detail coefficients" for each node.
        # So Level 0 node has 7 details. Level 1 nodes have 7 details.
        # Total 1 + 8*7 = 57 coefficients?
        # The paper mentions "all 15 coefficients must be retrieved".
        # This implies a specific reconstruction path.
        # Let's stick to the decoding cases.
        
        # If node_index is 0, we are looking for c0.
        # Case 3 doesn't apply to c0 in the same way?
        # The paper says "c0 ... need not to use the bytetag information".
        # So we just retrieve it from the one-byte stream.
        offset = unit_block.cell_info_blocks[cell_index].offset_one_byte
        # Displacement for c0 is 0? Or calculated?
        # "c0 ... stored in one-byte stream".
        # Let's assume c0 is the first element.
        value = unit_block.one_byte_stream[offset]
        return value
        
    else:
        # Detail coefficient
        # Get the significance map byte for this node
        # The significance maps start after the zerobits byte
        sig_maps = cell_info[1:]
        
        if sig_map_index >= len(sig_maps):
            # Should not happen if zerobit is 1
            return 0
            
        sig_map_byte = sig_maps[sig_map_index]
        
        # Case 3: Bit-flag in significance map is 0
        # The bit-flag corresponds to the position within the node's 7 coefficients.
        # Map (i', j', k') to a coefficient index within the node.
        # Each Level 1 node is 2x2x2.
        # Coefficient index within node = (i' % 2) * 4 + (j' % 2) * 2 + (k' % 2)
        coeff_index_in_node = (i_prime % 2) * 4 + (j_prime % 2) * 2 + (k_prime % 2)
        
        # Check the bit in sig_map_byte
        # Bits 1-7 are for coefficients. Bit 0 is two-byte flag.
        bit_flag = (sig_map_byte >> (coeff_index_in_node + 1)) & 1
        
        if bit_flag == 0:
            return 0
            
        # Case 4: Calculate address
        # Displacement = number of preceding non-zero coefficients with flag 1
        
        # 1. Count preceding zerobits that were 1 (preceding nodes)
        # These nodes' significance maps contribute to the stream.
        # We need to count the number of set bits in all preceding significance maps.
        total_displacement = 0
        
        # Count set bits in preceding significance maps
        for m in range(sig_map_index):
            total_displacement += count_set_bits(sig_maps[m])
            
        # 2. Count set bits in current significance map before current coefficient
        for b in range(1, coeff_index_in_node + 1):
            if (sig_map_byte >> b) & 1:
                total_displacement += 1
                
        # Determine stream (one-byte or two-byte)
        # Check two-byte flag (Bit 0 of sig_map_byte)
        two_byte_flag = sig_map_byte & 1
        
        # If two_byte_flag is 0, there are two-byte coefficients.
        # Check bytetag to see if THIS coefficient is two-byte.
        # The bytetag is stored in the cell info or stream header.
        # Assuming bytetag is available in cell_info
        bytetag = cell_info[1 + sig_map_index + 1] # Assuming bytetag follows sig maps?
        # The paper says "preserve a bytetag". It might be part of cell info.
        # Let's assume it's in the cell info block.
        
        is_two_byte = False
        if two_byte_flag == 0:
            # Check bytetag bit for this coefficient
            # Bytetag bit corresponds to coeff_index_in_node
            if (bytetag >> coeff_index_in_node) & 1:
                is_two_byte = True
                
        if is_two_byte:
            offset = unit_block.cell_info_blocks[cell_index].offset_two_byte
            value = unit_block.two_byte_stream[offset + total_displacement]
        else:
            offset = unit_block.cell_info_blocks[cell_index].offset_one_byte
            value = unit_block.one_byte_stream[offset + total_displacement]
            
        return value

def count_set_bits(byte):
    """Helper to count set bits using precomputed table T(*)"""
    # T is a global precomputed table
    return T[byte]
```

**Parameters:**
*   **Indexing Table T(*):** 256 entries. $T[x]$ returns the number of 1-bits in byte $x$.
*   **CT Data:** $c_0$ is stored in one-byte stream even if zero.
*   **RGB Data:** $c_0$ is a 3-component vector. Range $[0, 255]$. Detail vectors range $[-128, 127]$.

**Edge Cases and Pitfalls:**
*   **CT Data $c_0$:** Stored in one-byte stream even if zero, causing unnecessary storage overhead [S1, unofficial]. This is considered acceptable because $c_0$ being zero often implies the cell is null [S1, unofficial].
*   **RGB Data:** Requires 3-byte streams for $c_0$ and detail vectors, differing from CT data handling [S1, unofficial].
*   **Precision:** Coefficients are stored as signed/unsigned chars.

## Operating characteristics
*   **Compression Ratio:** Improves over previous schemes that used a fixed 8-byte significance map. The average number of bytes for the significance map in non-null cells is less than 8 bytes [S1, unofficial].
*   **Random Access Speed:** Supports fast random access by using the Cell Tag Table and zerobits to quickly skip zero coefficients [S1, unofficial].
*   **Memory Complexity:** Requires storage for Cell Tag Table (1 byte/cell), Cell Information (variable), and Byte Streams.
*   **Strengths:** High compression ratio, fast random access, no image quality loss [S1, unofficial].
*   **Limitations:** Slightly more complex decoding logic due to variable-length significance maps and two-byte streams.

## Background
The Zerobit Encoding Scheme was proposed in a paper presented at the Visible Human Project conference in 1998 [S1, unofficial]. It builds on previous wavelet compression schemes for volume data, aiming to improve compression ratio and random access speed by exploiting spatial coherence in wavelet coefficients [S1, unofficial].

## Sources
[S1] Compression Scheme, https://www.nlm.nih.gov/research/visible/vhpconf98/AUTHORS/PARK/SECTION3.HTM, fetched 2026-10-28 (unofficial)
