# Pointer Compression

Also called: Frequency-Based Pointer Encoding, Pointer Table Encoding

## Overview
Pointer compression is a sparse structure optimization that reduces the memory footprint of child pointers in tree-based representations (such as Sparse Voxel DAGs) by replacing absolute addresses with indices into a frequency-ordered lookup table [S1]. It sets itself apart from sibling methods like [Offset Compression](1819-offset-compression.md) by targeting the high-value pointers at upper tree levels, where absolute pointers consume the most memory, whereas offset compression targets the frequent small deltas in lower levels [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse voxel structure (e.g., DAG) containing child pointers, typically 32-bit absolute addresses.
*   **Output:** A compressed representation where pointers are stored as variable-length indices into a global or per-level lookup table.

**Data Structures:**
1.  **Pointer Table:** A list of absolute pointers, sorted by occurrence frequency in descending order. One table is maintained per tree level to maximize hit rates for common subtrees [S1].
2.  **Compressed Pointer Field:** A variable-length bit field stored in the node. It uses a 2-bit type header followed by the payload.

**Numbered Steps:**
1.  **Frequency Analysis:** Traverse the structure and count the occurrences of each unique child pointer value for each tree level.
2.  **Table Construction:** Sort the unique pointers for each level by frequency in descending order. The most frequent pointer is assigned index 0, the next index 1, etc. [S1].
3.  **Bit-Length Assignment:** Determine the bit length required for the index based on the table size and the chosen encoding scheme. The method uses a fixed set of payload sizes: 6, 14, or 22 bits for table indices, and 30 bits for absolute pointers [S1].
4.  **Encoding:** Replace each absolute pointer in the node with a compressed value:
    *   If the pointer exists in the table, store its index using the appropriate bit length (6, 14, or 22) indicated by the type header.
    *   If the pointer is not in the table (or the table is full), store the absolute pointer value using the 30-bit absolute type.
5.  **Decoding:** At runtime, read the 2-bit type header. If it indicates a table index, look up the absolute address in the level-specific pointer table. If it indicates an absolute pointer, use the remaining 30 bits directly.

**Formulas:**
Let $P$ be the absolute pointer value. Let $T_l$ be the pointer table for level $l$. Let $idx(P, T_l)$ be the index of $P$ in $T_l$.
The compressed value $C$ is structured as:
$$ C = \{ \text{type} \}_{2\text{-bits}} \parallel \{ \text{payload} \}_{k\text{-bits}} $$
Where $k \in \{6, 14, 22, 30\}$ and type mapping is:
*   Type 0: 6-bit index into $T_l$
*   Type 1: 14-bit index into $T_l$
*   Type 2: 22-bit index into $T_l$
*   Type 3: 30-bit absolute pointer $P$ [S1].

**Parameters:**
*   **Table Size:** Determined by the number of unique pointers per level. The sources note that 30-bit absolute pointers are sufficient for DAG nodes in tested examples [S1].
*   **Bit Lengths:** Fixed at 6, 14, 22, and 30 bits for the payload [S1].

**Edge Cases and Pitfalls:**
*   **Performance:** Entropy encoding decreases rendering performance due to the overhead of table lookups and variable-length decoding [S1].
*   **Memory vs. Speed:** While compression improves memory usage, the trade-off is increased computational cost during traversal [S1].
*   **Table Overflow:** If a pointer is not in the table, it falls back to the 30-bit absolute representation, which uses more bits than a 6-bit index but ensures correctness [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data structures
class PointerTable:
    def __init__(self):
        self.entries = []  # List of absolute pointers
        self.index_map = {}  # Map absolute pointer -> index

    def add_pointer(self, ptr):
        if ptr not in self.index_map:
            self.index_map[ptr] = len(self.entries)
            self.entries.append(ptr)
        return self.index_map[ptr]

    def get_pointer(self, index):
        return self.entries[index]

class CompressedPointer:
    def __init__(self, ptr, table, level):
        self.ptr = ptr
        self.table = table
        self.level = level
        self.compressed_bits = self._encode()

    def _encode(self):
        # Check if pointer is in table
        if self.ptr in self.table.index_map:
            idx = self.table.index_map[self.ptr]
            # Determine bit length based on index size and type
            # Note: The source specifies fixed types for 6, 14, 22 bits
            # In practice, one might choose the smallest type that fits the index
            # However, the source implies a specific setup was found effective.
            # We assume the encoder selects the type based on the index value
            # or a pre-defined mapping. For simplicity, we map index ranges to types.
            if idx < 64:  # 2^6
                return (0, idx, 6)
            elif idx < 16384:  # 2^14
                return (1, idx, 14)
            elif idx < 4194304:  # 2^22
                return (2, idx, 22)
            else:
                # Fallback to absolute if index is too large for 22 bits
                # Or if the source implies 22 is the max index size
                return (3, self.ptr, 30)
        else:
            # Not in table, store absolute
            return (3, self.ptr, 30)

    def decode(self):
        type_code, payload, bit_len = self.compressed_bits
        if type_code == 3:
            return payload
        else:
            return self.table.get_pointer(payload)

# Initialization
# 1. Traverse scene, count pointer frequencies per level
# 2. Build PointerTable for each level, sorted by frequency
# 3. Replace pointers in nodes with CompressedPointer objects
```

## Subtopics

### [Offset Compression](1819-offset-compression.md)
A related technique that compresses the offset from a node to its first child, exploiting the fact that offsets are often small and start at +1. It is typically applied alongside pointer compression to maximize memory savings in lower tree levels.

## Operating characteristics
*   **Memory Complexity:** Reduces the average bits per pointer from 32 to a weighted average of 6, 14, 22, or 30 bits, depending on the frequency distribution of pointers in the scene [S1].
*   **Performance:** Decreases rendering performance due to the overhead of entropy decoding and table lookups [S1].
*   **Strengths:** Significant memory reduction for scenes with repetitive subtrees or common pointer patterns [S1].
*   **Limitations:** Performance hit; less effective if pointer distribution is uniform or random [S1].

## Background
Pointer compression was introduced in the context of geometry and attribute compression for voxel scenes, specifically for Sparse Voxel DAGs [S1]. It builds on the observation that certain subtrees are used significantly more often than others, making entropy encoding a suitable candidate for memory reduction [S1]. The method was evaluated on architectural structures, complex geometry, and real-life 3D models [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
