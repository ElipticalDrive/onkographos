# PSVDAG Label/Caller Encoding with Frequency-Based Compaction

Also called: Level-wise label assignment, frequency-sorted caller encoding

## Overview
This method replaces the fixed 32-bit child node pointers used in Sparse Voxel Directed Acyclic Graphs (SVDAG) with variable-length binary labels (callers) to enable pointerless traversal while minimizing memory footprint [S1]. It sets itself apart from standard SVDAG by achieving significant compression (up to 5.33-fold) through frequency-based sorting and level-wise label space reset, at the cost of requiring label reconstruction during traversal [S1].

## How it works
**Inputs:** A PSVDAG structure where nodes are organized into levels, and each Inner Node (INODE) references child subgraph roots from the previous level.
**Outputs:** A compacted representation where child references are encoded as variable-length binary labels.

**Data Structures:**
*   **Label Table (per level):** A mapping from subgraph root node identifiers to binary codes.
*   **Binary Code:** Variable-length bit sequence. Minimum length is 6 bits.

**Steps:**
1.  **Collect References:** For each graph level $l$, identify all unique subgraph root nodes referenced by nodes at level $l-1$.
2.  **Count Frequency:** Count the number of times each unique subgraph root node is referenced from level $l-1$. Let $f_i$ be the frequency of the $i$-th unique node.
3.  **Sort Nodes:** Sort the unique subgraph root nodes in descending order of their frequency $f_i$.
4.  **Assign Labels:** Assign binary labels to the sorted nodes.
    *   The most frequent node receives the shortest label.
    *   The minimum label length is 6 bits [S1].
    *   Subsequent nodes receive longer labels as needed to maintain unique prefix-free codes (or simply sequential codes if prefix-free is not strictly enforced by the specific implementation, though "binary labels" implies distinct codes). The length increases gradually with the number of unique labels at that level [S1].
5.  **Reset Label Space:** Repeat steps 1-4 independently for each graph level. This resets the label assignment, ensuring that the most frequent nodes at *every* level get the shortest 6-bit codes, rather than competing for short codes globally [S1].

**Formulas:**
*   **Compression Ratio:** The shortest labels achieve a 5.33-fold compression rate compared to 32-bit pointers [S1].
    $$ \text{Compression Ratio} = \frac{32 \text{ bits}}{6 \text{ bits}} \approx 5.33 $$
*   **Space Occupancy:** The shortest labels occupy 18.75% of the space of a 32-bit pointer [S1].
    $$ \text{Space Occupancy} = \frac{6}{32} = 0.1875 = 18.75\% $$

**Parameters:**
*   **Minimum Label Length:** 6 bits [S1].
*   **SVDAG Pointer Size:** 32 bits [S1].

**Edge Cases and Pitfalls:**
*   **Unique Subgraphs:** If a subgraph is unique (referenced only once), the pointer in SVDAG is 32 bits, but in PSVDAG, if the node itself is not referenced by a caller, it may have 0 bits for caller encoding, but the reference *to* it still requires a label [S1].
*   **Precision:** The binary representation length ranges from 6 bits to 16 bits (or more, depending on the number of unique nodes per level) [S1]. The sum of auxiliary fields (ACHNC and CHNM) ranges from 5 to 19 bits, which is significantly less than SVDAG's fixed overhead [S1].
*   **Level-wise Reset:** Failing to reset the label space per level would force a global frequency count, potentially assigning long codes to frequently occurring nodes at deeper levels if they are less frequent globally [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_labels_per_level(level_nodes, previous_level_references):
    """
    Encodes child references for a specific level using frequency-based compaction.
    
    Args:
        level_nodes: List of nodes at the current level.
        previous_level_references: List of subgraph root nodes referenced by level_nodes.
        
    Returns:
        label_map: Dictionary mapping node_id to binary string.
    """
    # Step 1 & 2: Count frequency of each referenced subgraph root
    frequency_map = {}
    for ref_node_id in previous_level_references:
        frequency_map[ref_node_id] = frequency_map.get(ref_node_id, 0) + 1
        
    # Step 3: Sort nodes by frequency (descending)
    sorted_nodes = sorted(frequency_map.keys(), key=lambda x: frequency_map[x], reverse=True)
    
    # Step 4: Assign binary labels
    label_map = {}
    current_code = 0
    current_length = 6  # Minimum label length
    
    for i, node_id in enumerate(sorted_nodes):
        # Calculate required length for unique code
        # Assuming prefix-free or simple binary counting, length increases as needed
        # The source states length increases gradually with the number of unique labels
        # We determine the bit length required to represent the index 'i' uniquely
        # starting from a base of 6 bits.
        
        # Simple approach: assign sequential binary codes, increasing length when necessary
        # to ensure uniqueness. The source implies a variable-length code scheme.
        # A common implementation for variable-length codes with a minimum length is
        # to use a prefix code or simply increase the bit width.
        
        # Determine bit width needed for index i
        # If i < 2^6, 6 bits suffice. If i >= 2^6, we need more bits.
        # However, the source says "starting with the shortest representation (6 bits)
        # for the most frequent nodes, and extending the length for less frequent nodes."
        # This suggests a Huffman-like or exponential-Golomb-like behavior, or simply
        # increasing the bit-width for the next block of nodes.
        
        # Let's assume a simple variable-length encoding where we increment length
        # when the number of nodes exceeds 2^length.
        
        bits_needed = max(6, (i.bit_length() if i > 0 else 1))
        # Ensure we don't drop below 6 bits
        if bits_needed < 6:
            bits_needed = 6
            
        # Encode the index 'i' into 'bits_needed' bits
        # Note: This is a simplified assignment. A true prefix-free code would be more complex.
        # The source does not specify the exact coding scheme (e.g., Huffman, Exp-Golomb),
        # only that it is variable-length and frequency-sorted.
        # We will use a simple binary representation of the rank 'i' padded to bits_needed.
        
        binary_code = format(i, f'0{bits_needed}b')
        label_map[node_id] = binary_code
        
    return label_map
```

## Subtopics

### Pointerless SVO
A compact, pointerless structure suitable for archival/streaming, but lacks fast traversal capabilities [S1].

### SVDAG
Uses 32-bit pointers for fast traversal and subtree reduction, but has higher memory overhead [S1].

### PSVDAG
Combines the compactness of Pointerless SVO with the traversal efficiency of SVDAG by using variable-length labels [S1].

## Operating characteristics
*   **Time Complexity:** Encoding requires sorting nodes by frequency per level, which is $O(N \log N)$ where $N$ is the number of unique references per level. Traversal requires label reconstruction, adding overhead compared to direct pointer dereferencing [S1].
*   **Memory Complexity:** Significantly reduced compared to SVDAG. The average label/caller size ranges from 0.625 to 2.375 bits per child node, depending on the scene and resolution [S1].
*   **Measured Performance:** On the tested scenes, the compression ratio ranges from 1.68 to 6.4 compared to SVDAG [S1]. The reduction in data structure size is between 15.63% and 59.38% [S1].
*   **Strengths:** High compression ratio for scenes with frequent subgraph reuse; pointerless traversal enables efficient streaming/archival [S1].
*   **Limitations:** Requires label reconstruction during traversal, which may be slower than direct pointer access; performance depends on the frequency distribution of subgraphs [S1].
*   **When to Use:** Ideal for archival, streaming, or memory-constrained environments where subgraph reuse is high [S1].
*   **When Not to Use:** When fast, direct traversal is critical and memory is not a constraint, SVDAG may be preferable [S1].

## Background
The PSVDAG structure was proposed by Backus et al. in 2020 [S1]. It builds on the advantages of both Pointerless SVO and SVDAG, aiming to provide a compact encoding suitable for 3D scene representation [S1]. The method addresses the need for efficient storage and traversal in voxelized 3D scenes [S1].

## Sources
[S1] PSVDAG: Compact Voxelized Representation of 3D Scenes Using Pointerless Sparse Voxel Directed Acyclic Graphs, https://doi.org/10.31577/cai_2020_3_587, fetched 2026-10-26
