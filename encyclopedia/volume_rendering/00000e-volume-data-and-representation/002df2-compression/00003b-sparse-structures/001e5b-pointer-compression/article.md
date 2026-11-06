# Pointer Compression

Also called: Frequency-Based Pointer Encoding, Pointer Table Encoding

## Overview
Pointer compression is a sparse structure optimization that reduces the memory footprint of child pointers in tree-based representations (such as Sparse Voxel DAGs) by replacing absolute addresses with indices into a frequency-ordered lookup table [S1]. It sets itself apart from sibling methods like [Offset Compression](1819-offset-compression.md) by targeting frequent subtree references across tree levels, though it yields less compression than offset encoding because the majority of pointers reside at levels that still require full four-byte storage [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse voxel structure (e.g., DAG) containing child pointers, typically 32-bit absolute addresses [S1].
*   **Output:** A compressed representation where pointers are stored as variable-length indices into a per-level lookup table [S1].

**Data Structures:**
1.  **Pointer Table:** A list of absolute pointers sorted by occurrence frequency in descending order, maintained per tree level to maximize hit rates for common subtrees [S1].
2.  **Compressed Pointer Field:** A 1-byte field per pointer, structured as a 2-bit type header followed by a variable-length payload [S1].

**Numbered Steps:**
1.  **Frequency Analysis:** Traverse the structure and count occurrences of each unique child pointer value for each tree level [S1].
2.  **Table Construction:** Sort the unique pointers for each level by frequency in descending order to build the pointer table [S1].
3.  **Bit-Length Assignment:** Assign one of three fixed payload sizes (6, 14, or 22 bits) for table indices, or 30 bits for absolute pointers, ensuring byte alignment [S1].
4.  **Encoding:** Replace each absolute pointer with a compressed value: if the pointer exists in the table, store its index using the appropriate bit length indicated by the type header; if not, store the absolute pointer value using the 30-bit absolute type [S1].
5.  **Decoding:** At runtime, read the 2-bit type header; if it indicates a table index, look up the absolute address in the level-specific pointer table; if it indicates an absolute pointer, use the remaining 30 bits directly [S1].

**Formulas:**
Let $P$ be the absolute pointer value. Let $T_l$ be the pointer table for level $l$. Let $idx(P, T_l)$ be the index of $P$ in $T_l$.
The compressed value $C$ is structured as:
$$ C = \{ \text{type} \}_{2\text{-bits}} \parallel \{ \text{payload} \}_{k\text{-bits}} $$
Where $k \in \{6, 14, 22, 30\}$ and the type mapping is:
*   Type 0: 6-bit index into $T_l$
*   Type 1: 14-bit index into $T_l$
*   Type 2: 22-bit index into $T_l$
*   Type 3: 30-bit absolute pointer $P$ [S1].

**Parameters:**
*   **Table Size:** Determined by the number of unique pointers per level; the sources note that 30-bit absolute pointers are sufficient for DAG nodes in tested examples [S1].
*   **Bit Lengths:** Fixed at 6, 14, 22, and 30 bits for the payload [S1].

**Edge Cases and Pitfalls:**
*   **Performance:** Entropy encoding decreases rendering performance [S1].
*   **Compression Efficiency:** The technique yields less compression than offset encoding because the levels containing most pointers generally require the full four bytes per pointer [S1].
*   **Fallback:** Pointers not in the table or exceeding index capacity fall back to the 30-bit absolute representation, ensuring correctness at the cost of higher bit usage [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_pointer_table(level_pointers):
    freq = {}
    for ptr in level_pointers:
        freq[ptr] = freq.get(ptr, 0) + 1
    sorted_ptrs = sorted(freq.keys(), key=lambda p: freq[p], reverse=True)
    return {ptr: i for i, ptr in enumerate(sorted_ptrs)}

def encode_pointer(ptr, table):
    if ptr in table:
        idx = table[ptr]
        if idx < 64:
            return (0, idx, 6)
        elif idx < 16384:
            return (1, idx, 14)
        elif idx < 4194304:
            return (2, idx, 22)
    return (3, ptr, 30)

def decode_pointer(compressed, table):
    type_code, payload, _ = compressed
    if type_code == 3:
        return payload
    return table[payload]
```

## Subtopics

### [Offset Compression](1819-offset-compression.md)
A related technique that compresses the offset from a node to its first child, exploiting the fact that offsets are often small and start at +1. It is typically applied alongside pointer compression to maximize memory savings in lower tree levels.

## Operating characteristics
*   **Memory Complexity:** Reduces the average bits per pointer from 32 to a weighted average of 6, 14, 22, or 30 bits, depending on the frequency distribution of pointers in the scene [S1].
*   **Performance:** Decreases rendering performance due to the overhead of entropy decoding [S1].
*   **Strengths:** Significant memory reduction for scenes with repetitive subtrees or common pointer patterns [S1].
*   **Limitations:** Yields less compression than offset encoding because most pointers reside at levels requiring full four-byte storage [S1].

## Background
Pointer compression was introduced in the context of geometry and attribute compression for voxel scenes, specifically for Sparse Voxel DAGs [S1]. It builds on the observation that certain subtrees are used significantly more often than others, making entropy encoding a suitable candidate for memory reduction [S1]. The method was evaluated on architectural structures, complex geometry, and real-life 3D models [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
