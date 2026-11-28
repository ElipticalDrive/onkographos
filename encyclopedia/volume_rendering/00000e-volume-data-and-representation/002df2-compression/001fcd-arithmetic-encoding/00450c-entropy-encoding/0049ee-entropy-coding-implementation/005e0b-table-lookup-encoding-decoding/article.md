# Table-Lookup Encoding/Decoding
Also called: Direct Lookup Table Method
## Overview
Table-Lookup Encoding/Decoding is a method for implementing entropy codes by replacing complex parsing logic with direct memory access to a precomputed table [S1, unofficial]. It solves the problem of computational overhead in parse-tree-based codes (such as BV, VB, and VV codes) by allowing the encoder or decoder to determine the consumed input bits and the resulting codeword in a single memory lookup [S1, unofficial]. It differs from its siblings by prioritizing speed through cache-friendly table sizes over the theoretical optimality of variable-length parsing without fixed windows, and it avoids the state-tracking complexity of arithmetic coding [S1, unofficial].
## How it works
**Inputs:** An input bit stream and a window size parameter $W$.
**Outputs:** An output bit stream containing the encoded codewords (encoding) or the decoded symbols (decoding).

**Data Structures:**
*   **Lookup Table:** An array of size $2^W$. Each entry is a fixed-size record (e.g., 32 bits) containing three fields:
    *   $l$: The number of bits consumed from the input stream for this specific codeword.
    *   $c$: The length (in bits) of the codeword produced.
    *   `code`: The actual bit pattern of the codeword.

**Parameters:**
*   $W$: The window size in bits. It must be at least as long as the longest string on the parse tree (for encoding) or the longest codeword (for decoding) [S1, unofficial].
*   Table Entry Size: Typically 32 bits, though this affects cache performance [S1, unofficial].

**Algorithm Steps:**
1.  Read the next $W$ bits from the input stream.
2.  Use these $W$ bits as an index to access the corresponding entry in the lookup table.
3.  Retrieve $l$, $c$, and `code` from the table entry.
4.  Advance the input stream pointer by $l$ bits.
5.  Advance the output stream pointer by $c$ bits.
6.  Write `code` to the output stream at the new position.
7.  Repeat until the input stream is exhausted.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def table_lookup_encode(input_bits, lookup_table, W):
    """
    Encodes input_bits using a precomputed lookup table.
    
    Args:
        input_bits: A list or array of bits (0 or 1).
        lookup_table: A list of tuples (l, c, code), indexed by W-bit integer.
        W: The window size in bits.
        
    Returns:
        A list of bits representing the encoded output.
    """
    output_bits = []
    input_len = len(input_bits)
    
    # Process input in chunks of W bits
    for i in range(0, input_len, W):
        # Extract the next W bits. 
        # If the input ends before W bits, pad with zeros or handle as end-of-stream marker.
        # For simplicity, assume input length is a multiple of W or handled by logic.
        chunk_start = i
        chunk_end = min(i + W, input_len)
        
        # Form the index from the W bits
        # Note: In practice, bit ordering (MSB/LSB) must be consistent.
        # Here we assume the first bit in the chunk is the MSB of the index.
        index = 0
        for j in range(chunk_start, chunk_end):
            index = (index << 1) | input_bits[j]
            
        # If we didn't get a full W bits, this is an edge case.
        # Standard implementation might require padding or a specific EOF handling.
        # Assuming valid input where W bits are always available or handled by a sentinel.
        if chunk_end - chunk_start < W:
            # Handle incomplete window (e.g., EOF)
            # This depends on specific code design. 
            # For this generic description, we assume W-bit alignment or a specific EOF code.
            break 
            
        # Access the table
        entry = lookup_table[index]
        l, c, code = entry
        
        # Shift input by l bits (already handled by loop incrementing by W, 
        # but logically l bits are consumed from the current window context)
        # The loop structure above assumes W-bit fixed steps. 
        # A more precise implementation tracks bit position:
        
        # Corrected logic for variable l consumption:
        # The loop should not necessarily step by W, but by l. 
        # However, the lookup requires W bits to *predict* l.
        # Standard implementation:
        # 1. Look at next W bits.
        # 2. Determine l.
        # 3. Consume l bits.
        # 4. Repeat.
        
        # Re-writing loop for variable consumption:
        pass

def table_lookup_encode_precise(input_bits, lookup_table, W):
    output_bits = []
    input_pos = 0
    input_len = len(input_bits)
    
    while input_pos < input_len:
        # 1. Examine next W bits
        # Ensure we have W bits available. If not, handle EOF.
        if input_pos + W > input_len:
            # Handle end of stream. 
            # Often a specific codeword is reserved for EOF.
            # For this generic method, we assume the input is padded or 
            # the last codeword is handled separately.
            break
            
        # Extract W bits starting at input_pos
        index = 0
        for k in range(W):
            index = (index << 1) | input_bits[input_pos + k]
            
        # 2. Lookup table entry
        entry = lookup_table[index]
        l, c, code = entry
        
        # 3. Shift l bits off input
        input_pos += l
        
        # 4. Place codeword on output
        # code is a bit pattern. We need to append it to output_bits.
        # Assuming code is an integer of length c.
        for k in range(c):
            # Extract bits from code. Assuming MSB first.
            bit = (code >> (c - 1 - k)) & 1
            output_bits.append(bit)
            
    return output_bits
```

**Edge Cases and Pitfalls:**
*   **Table Size:** The table size grows exponentially with $W$ ($2^W$ entries). If the longest code length is excessively long, the table becomes too large for cache [S1, unofficial]. For example, with 32-bit entries, a 16-bit code length yields a 256 KB table, while a 20-bit length yields a 4 MB table, a 1600% increase [S1, unofficial].
*   **Worst Case:** If there are $N$ possible inputs, the longest possible code length is $N-1$ bits, making the table size unmanageable [S1, unofficial].
*   **Cache Misses:** If the table exceeds cache capacity, performance degrades significantly due to memory latency [S1, unofficial].
*   **Input Alignment:** The input stream must be processed in a way that allows reading $W$ bits for lookup, even if only $l$ bits are consumed. This requires buffering or bit-shifting logic to maintain the window [S1, unofficial].

## Subtopics

### [Bin Coder](17560-bin-coder.md)
The Bin Coder is a zero-order entropy coding method that partitions a memoryless source into multiple sub-sources (bins) based on bit probability, encoding each with a specialized coder optimized for that specific probability range. It solves the problem of efficiently encoding nonstationary or skewed bit streams by avoiding the computational overhead of arithmetic coding while maintaining high compression ratios through specialized codes (VV and Golomb). It differs from State-Tree Codes by dividing bits by probability rather than switching codes after parsing a string, and from Boliek et al.'s hardware bin coder by eliminating output interleaving overhead.

## Operating characteristics
*   **Time Complexity:** Encoding and decoding are $O(1)$ per codeword, dominated by memory access and bit manipulation. The coding loop is described as "very tight" [S1, unofficial].
*   **Memory Complexity:** $O(2^W)$, where $W$ is the window size. Each entry typically requires 32 bits [S1, unofficial].
*   **Performance:** Extremely fast due to minimal computational overhead, provided the lookup table fits in cache [S1, unofficial].
*   **Strengths:** High speed, simple implementation, low computational overhead compared to arithmetic coding or complex parsing [S1, unofficial].
*   **Limitations:** High memory usage for large $W$; table size explosion with long codewords; requires precomputation of the table [S1, unofficial].
*   **When to Use:** When speed is critical and the code lengths are short enough to fit the lookup table in cache (e.g., $W \le 16$) [S1, unofficial].
*   **When Not to Use:** When memory is constrained, or when code lengths are very long (e.g., $W > 20$), leading to cache misses or excessive memory usage [S1, unofficial].

## Background
The method was explored in the context of length-limited data transformation and compression, specifically for parse-tree-based codes like BV, VB, and VV [S1, unofficial]. The author noted that while arithmetic codes provide the best compression rates, they are too slow, and run-length codes (like Golomb) are fast but only efficient under specific probability distributions [S1, unofficial]. The table-lookup method was designed to achieve a balance between speed and compression efficiency by minimizing computational overhead [S1, unofficial].

## Sources
[S1] Length-Limited Data Transformation and Compression, https://digital.library.unt.edu/ark:/67531/metadc874691/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
