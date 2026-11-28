# Z-order Index Remapping for Binary Tree Hierarchy

Also called: 2D/3D Z-order interleaving, Lebesgue space filling curve indexing

## Overview
A method to map a Z-order (Lebesgue) 1D index to a hierarchical out-of-core binary tree index, allowing efficient traversal of rectilinear grids by determining the element's level and local index within that level [S1, unofficial]. It builds on [Global Static Indexing for Regular Grids](21466-global-static-indexing-for-regular-grids.md) by providing the specific indexing logic required to traverse the binary tree hierarchy associated with the Z-order storage layout, enabling out-of-core visualization where data is accessed based on resolution levels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A Z-order index $j$ (integer) representing an element's position in the 1D Lebesgue curve, and the total number of hierarchy levels $k$.
- Output: A hierarchical index $I'(j)$ that identifies the element's position within the binary tree structure (level $i$ and local offset).

**Data Structures:**
- The Z-order index $j$ is stored as a standard integer.
- The hierarchy is defined by $k$ levels, where level 0 is the coarsest and level $k-1$ is the finest.

**Formulas and Definitions:**
1.  **Z-order Index Construction:** The 1D Z-order index $I$ for a $d$-dimensional coordinate $(i_1, \dots, i_d)$ is formed by interleaving the bits of the binary representations of the coordinates. If $i_j$ is represented by $h$ bits $b_j^1 \dots b_j^h$, then:
    $$ I = b_1^1 b_1^2 \dots b_1^d b_2^1 \dots b_2^d \dots b_h^1 \dots b_h^d $$
    [S1, unofficial]

2.  **Level Determination:** For a binary hierarchy with $k$ levels, the level $i$ of an element with Z-order index $j$ is determined by the number of trailing zeros $h$ in the binary representation of $j$:
    $$ i = k - h $$
    [S1, unofficial]

3.  **Coarse Offset ($C_i$):** The constant $C_i$ representing the number of elements in levels coarser than $i$ is:
    $$ C_i = 2^{i-1} \quad \text{for } i > 0, \quad C_0 = 0 $$
    [S1, unofficial]

4.  **Local Index ($I''_i$):** The local index $I''_i(j)$ for an element with Z-order index $j$ at level $i$ is:
    $$ I''_i(j) = \left\lfloor \frac{j}{2^{i+1}} \right\rfloor $$
    [S1, unofficial]

5.  **Final Hierarchical Index:**
    $$ I'(j) = C_i + I''_i(j) $$
    [S1, unofficial]

**Algorithm Steps:**
The remapping can be performed efficiently using bitwise operations. The algorithm sets the leftmost bit (position $k+1$) to 1, then shifts the index right until a 1 comes out of the bit-string, effectively removing trailing zeros and the leading 1.

1.  Set the bit at position $k+1$ (the most significant bit for the given depth) to 1. This ensures the shift loop terminates correctly.
2.  Remove all trailing zeros from the index. This is done by dividing the index by its lowest set bit (LSB).
3.  Shift the result right by 1 bit to remove the leading 1 set in step 1.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def remap_zorder_to_hierarchical(z_index: int, k: int) -> int:
    """
    Remaps a Z-order index to a hierarchical binary tree index.
    
    Args:
        z_index: The 1D Z-order index (integer).
        k: The total number of levels in the binary hierarchy.
        
    Returns:
        The hierarchical index I'(s).
    """
    # Step 1: Set the bit in position k + 1 (leftmost bit for depth k)
    # Assuming z_index fits within k bits, we set the (k+1)-th bit.
    # In Python, integers have arbitrary precision, so we mask appropriately if needed.
    # For a k-level tree, indices are typically 0 to 2^k - 1.
    # The algorithm description implies setting the bit at index k (0-indexed from LSB)
    # which corresponds to value 2^k.
    z_index |= (1 << k)
    
    # Step 2: Remove trailing zeros.
    # This is equivalent to dividing by the lowest set bit.
    # Portable way to get LSB: x & (~x + 1)
    lsb = z_index & (~z_index + 1)
    z_index //= lsb
    
    # Step 3: Shift right by 1 to remove the leading 1 set in Step 1.
    return z_index >> 1
```
[S1, unofficial]

**Edge Cases and Pitfalls:**
-   **Two's Complement:** The C++ implementation `i /= i&-i` relies on two's complement representation for the negation `-i`. For portability, replace `-i` with `(~i)+1` [S1, unofficial].
-   **Precision:** The algorithm is efficient for runtime execution but requires integer arithmetic. Ensure the Z-order index fits within the integer type used.
-   **Level 0:** For level 0, $C_0 = 0$. The formula for $I''_0(j)$ becomes $\lfloor j/2 \rfloor$. The algorithm should handle $k=0$ or $i=0$ correctly if such cases arise, though typically $k \ge 1$.

## Subtopics

### Z-order Index Remapping for $2^l$ Tree Hierarchy
A generalized index remapping scheme for $2^l$ trees (e.g., quad-tree where $l=2$, oct-tree where $l=3$) to align data layout with larger disk blocks, reducing the number of hierarchy levels compared to the binary tree scheme [S1, unofficial]. Unlike the binary tree scheme which shifts 1 bit at a time, this scheme shifts the bit-string in blocks of $l$ bits to align with $2^l$ tree structures [S1, unofficial].
-   **Level Determination:** Level $i$ is determined by the number of trailing sequences of $l$ zeros at the end of the Z-order index bit-string [S1, unofficial].
-   **Coarse Offset ($C_i$):** $C_i = (2^l)^{i-1}$ for $i > 0$, and $C_0 = 0$ [S1, unofficial].
-   **Local Index ($I''_i$):** $I''_i(j) = \lfloor \frac{j}{2^{l \cdot i}} \rfloor - \lfloor \frac{j}{2^{l(i+1)}} \rfloor - 1$ [S1, unofficial].
-   **Final Index:** Computed by shifting $I(s)$ right and adding the complement of its high bits, leveraging $C_i = 2^{l(i-1)}$ [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The algorithm allows computing the hierarchical index efficiently at runtime, suitable for out-of-core storage where repeated execution is required [S1, unofficial]. The bitwise operations are $O(1)$ relative to the number of bits, or $O(\log N)$ where $N$ is the number of elements.
-   **Memory Complexity:** Minimal additional memory is required, as the remapping is computed on-the-fly from the Z-order index [S1, unofficial].
-   **Strengths:** Efficient for out-of-core traversal, aligns well with disk I/O patterns for rectilinear grids, and provides a direct mapping from spatial coordinates to hierarchical storage positions [S1, unofficial].
-   **Limitations:** The binary tree hierarchy may not align perfectly with larger disk block sizes, which is addressed by the $2^l$ variant [S1, unofficial].

## Background
The method originates from research on multi-resolution indexing for hierarchical out-of-core traversal of rectilinear grids [S1, unofficial]. It is based on the Lebesgue space filling curve, also known as the Z-order curve, which provides a hierarchical structure suitable for spatial indexing [S1, unofficial]. The approach is particularly useful for visualizing very large grids (e.g., $8192^3$) with modest memory by enabling out-of-core access patterns [S1, unofficial].

## Sources
[S1] Multi-Resolution Indexing for Hierarchical Out-of-Core Traversal of Rectilinear Grids, https://digital.library.unt.edu/ark:/67531/metadc724429/?q=out-of-core volume rendering, fetched 2026-10-28 (unofficial)
