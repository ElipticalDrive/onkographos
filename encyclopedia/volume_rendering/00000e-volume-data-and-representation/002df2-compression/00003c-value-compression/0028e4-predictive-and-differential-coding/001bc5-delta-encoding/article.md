# Delta Encoding

Also called: Heightfield Compression

## Overview
Delta encoding is a value compression technique that exploits the progressive variation of height values in volume data by storing differences between adjacent elements rather than absolute values [S1, unofficial]. It lacks the random access capabilities of its child, Block-Based Delta Encoding, which partitions data to mitigate the sequential access bottleneck [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of height values $H = [h_0, h_1, \dots, h_{N-1}]$ representing a heightfield or gstack interval [S1, unofficial].
*   **Output:** A compressed sequence of delta values $D = [d_0, d_1, \dots, d_{N-1}]$ [S1, unofficial].

**Data Structures:**
The output is a contiguous stream of signed integers representing the difference between consecutive heights [G].

**Steps:**
1.  **Initialize:** Set the first delta value $d_0$ equal to the first height value $h_0$ [G].
2.  **Compute Differences:** For each subsequent index $i$ from $1$ to $N-1$:
    $$d_i = h_i - h_{i-1}$$
3.  **Store:** Write $d_0, d_1, \dots, d_{N-1}$ to the output buffer [G].

**Parameters:**
*   **Bit Depth:** Not given in the sources for the base method [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Random Access Limitation:** Decompressing any single value $h_i$ requires reading and summing all preceding deltas $d_0 \dots d_i$, necessitating decompressing the whole dataset for any data access, which limits practical use in applications requiring efficient queries or traversals (e.g., real-time visualization) [S1, unofficial].
*   **Precision:** Accumulated errors can occur if floating-point deltas are summed, though integer heights are typical for heightfields [G].
*   **Boundary:** The first element $h_0$ must be preserved exactly, either as $d_0$ or in a header [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def delta_encode(heights):
    """
    Compresses a sequence of height values using delta encoding.
    
    Args:
        heights: List of integers representing height values [h0, h1, ..., hN-1]
        
    Returns:
        List of integers representing delta values [d0, d1, ..., dN-1]
    """
    if not heights:
        return []
        
    deltas = [0] * len(heights)
    deltas[0] = heights[0]
    
    for i in range(1, len(heights)):
        deltas[i] = heights[i] - heights[i-1]
        
    return deltas

def delta_decode(deltas):
    """
    Decompresses a sequence of delta values back to height values.
    
    Args:
        deltas: List of integers representing delta values [d0, d1, ..., dN-1]
        
    Returns:
        List of integers representing height values [h0, h1, ..., hN-1]
    """
    if not deltas:
        return []
        
    heights = [0] * len(deltas)
    heights[0] = deltas[0]
    
    for i in range(1, len(deltas)):
        heights[i] = heights[i-1] + deltas[i]
        
    return heights
```

## Subtopics

### [Heightfield Compression with Block-Based Delta Encoding](1116-heightfield-compression-with-block-based-delta-encoding.md)
This method compresses heightfields by partitioning them into equal-sized blocks, where each block is encoded relative to its minimum height value using a reduced bit-depth for the differences. It solves the primary limitation of standard delta encoding—where accessing any single value requires decompressing the entire dataset—by enabling random access with only two disk reads: one for the block header and one for the block's data. Unlike its parent method, Delta Encoding, which decouples interval heights in gstacks but lacks efficient random access, this approach balances compression ratio and access time, making it suitable for applications requiring efficient queries or traversals, such as real-time visualization.

### [Differences-based Compression](5723-differences-based-compression.md)
Differences-based Compression is a variable-bit-depth block compression scheme that stores compact differences relative to a block base value, expanding them to 16-bit precision during decompression using pre-computed shuffle patterns. It distinguishes itself from fixed-rate block coders (like VTC) by supporting variable bits-per-element per block and from simple delta encoding by using parallel SIMD expansion of stored differences rather than sequential accumulation.

## Operating characteristics
*   **Time Complexity:** Encoding is $O(N)$ where $N$ is the number of elements [G]. Decoding is $O(N)$ for full reconstruction, and $O(N)$ to access any single element, making random access inefficient [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the compressed deltas [G].
*   **Strengths:** Simple to implement; effective when height values vary progressively [S1, unofficial].
*   **Limitations:** Poor random access performance; requires sequential decompression for any query [S1, unofficial].
*   **When to use:** When data is accessed sequentially or stored in a way that allows full decompression (e.g., gstacks in offline processing) [S1, unofficial].
*   **When not to use:** When real-time visualization or efficient random access queries are required [S1, unofficial].

## Background
Delta encoding is suggested as a simple method providing good results because height values usually vary progressively [S1, unofficial]. It is used in the context of QuadStack, where attribute and heightfield representation are decoupled, allowing heightfields to be compressed by existing methods such as delta encoding [S1, unofficial]. The method is inspired by the work of Andujar [56] in the context of block-based variants [S1, unofficial].

## Sources
[S1] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
