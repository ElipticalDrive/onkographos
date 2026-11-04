# Pointerless Sparse Voxel Directed Acyclic Graph (PSVDAG)

Also called: Pointerless Sparse Voxel DAG

## Overview
The Pointerless Sparse Voxel Directed Acyclic Graph (PSVDAG) is a hierarchical data structure that represents a 3D voxel grid as a directed acyclic graph without using explicit pointers to child nodes [S1]. It combines the space-saving benefits of Pointerless Sparse Voxel Octrees (PSVOs) with the Common Subtree Merging (CSM) capabilities of Sparse Voxel DAGs (SVDAGs) [S2]. By replacing fixed-width pointers with variable-length binary labels and callers, PSVDAG achieves higher compression ratios than both SVOs and SVDAGs, making it suitable for archiving and streaming [S2]. However, the absence of direct pointers makes immediate traversal more complex compared to pointer-based structures [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid $R$ of size $N \times N \times N$ voxels, where $N = 2^n$ and $n \in \mathbb{N}$. Each voxel is binary: active ('1') or passive ('0') [S1].
*   **Output:** A binary stream representing the PSVDAG structure, organized into $n$ levels [S1].

**Data Structures and Layout:**
The structure is defined by Backus-Naur Form (BNF) [S1]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```bnf
PSVDAG ::= <NODE>
NODE ::= <INODE> | <LNODE>
INODE ::= <ACHNC><CHNM>
ACHNC ::= (3)<BIT>
CHNM ::= (1)*(8)<HT>
HT ::= "00" | "01"<LAB><NODE> | "10"<CAL> | "11"<NODE>
LAB ::= <SIZ><VAL>
CAL ::= <SIZ><VAL>
SIZ ::= (5)<BIT>
VAL ::= (1)*(32)<BIT>
LNODE ::= (8)<BIT>
BIT ::= "0" | "1"
```
*   **Levels:** Nodes are stored in $n$ levels, indexed by $l \in \{0, \dots, n-1\}$. The root is at $l=0$. Inner nodes are at $l \le n-2$. Leaf nodes are at $l = n-1$ [S1].
*   **Inner Node (INODE):**
    *   **ACHNC (Active Child Node Count):** A 3-bit unsigned integer storing the number of active children minus one ($c-1$). Range 0–7, representing 1–8 active children [S1].
    *   **CHNM (Child Node Mask):** A sequence of 1 to 8 Header Tags (HT). Passive children (HT '00') appearing after the last active child are omitted to save space [S1].
*   **Header Tags (HT):** 2-bit identifiers for child references [S1]:
    *   `'00'`: Passive child (empty). No further data follows.
    *   `'01'`: Label (LAB) child. The node is a unique template. Followed by a Label and the full binary representation of the child node.
    *   `'10'`: Caller (CAL) child. The node is a non-unique reference. Followed by a Caller.
    *   `'11'`: Direct child. The node is unique. Followed by the full binary representation of the child node.
*   **Label (LAB) and Caller (CAL):** Both have the structure `<SIZ><VAL>` [S1].
    *   **SIZ:** 5-bit unsigned integer (0–31). Represents the length of VAL minus 1.
    *   **VAL:** Variable-length bit vector of length $SIZ + 1$ bits. Range $0$ to $2^{SIZ+1}-1$. If $SIZ=0$, VAL is 1 bit [S1].
    *   **Uniqueness:** Labels and Callers are unique within a specific level $l$. The level index is not stored but derived during traversal [S1]. A Caller references the same node as a Label if they share the same level, SIZ, and VAL [S1].
*   **Leaf Node (LNODE):** An 8-bit vector representing the 8 voxels in a $2 \times 2 \times 2$ octant. Bit 0 is passive, Bit 1 is active. Bits are ordered according to a space-filling curve (z-order or Hilbert) [S1].

**Algorithm: Construction and Compaction**
The construction follows the bottom-up reduction principles of SVDAGs [Parent], with specific adaptations for pointerless encoding [S1, S2]:

1.  **Voxelization:** Convert the input geometry into a binary voxel grid $R$ of size $2^n \times 2^n \times 2^n$ [S2].
2.  **Tree Construction:** Build a standard Sparse Voxel Octree (SVO) from the grid. Identify common subtrees to form a Sparse Voxel DAG (SVDAG) using bottom-up reduction [Parent].
3.  **Frequency Analysis:** For each level $l$ from $n-2$ down to 0:
    *   Identify all nodes at level $l$ that are referenced more than once from level $l-1$. Let this set be $NOD$ [S1].
    *   Count the reference frequency for each node in $NOD$.
    *   Sort $NOD$ in descending order of frequency: $NOD = \{nod_0, nod_1, \dots, nod_{p-1}\}$ where $freq(nod_i) \ge freq(nod_{i+1})$ [S1].
4.  **Label Assignment (Frequency-Based Compaction):**
    *   Generate a set of labels $LAB = \{lab_0, lab_1, \dots, lab_{p-1}\}$ sorted by ascending binary length [S1].
    *   Assign the shortest label $lab_i$ to the most frequent node $nod_i$ [S1].
    *   The label $lab_i$ consists of $SIZ_i$ and $VAL_i$. The first label is $SIZ=0, VAL=0$. Subsequent labels increment $VAL$ until overflow, then increment $SIZ$ [S1].
5.  **Binary Encoding:**
    *   Traverse the SVDAG in Depth-First order [S1].
    *   For each node at level $l$:
        *   If the node is unique (referenced once), encode it with HT `'11'` followed by its binary representation [S1].
        *   If the node is non-unique:
            *   The first reference (template) uses HT `'01'`, followed by the assigned Label $LAB$ and the node's binary representation [S1].
            *   Subsequent references use HT `'10'`, followed by a Caller $CAL$ with the same $SIZ$ and $VAL$ as the assigned Label [S1].
        *   Passive children use HT `'00'` [S1].
    *   Omit trailing passive children from the CHNM sequence [S1].
    *   Prepend the ACHNC to each Inner Node [S1].
6.  **Leaf Encoding:** At level $n-1$, encode Leaf Nodes as 8-bit vectors using the selected space-filling curve [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_psvdag(voxel_grid, n):
    """
    Encodes a 3D voxel grid into a PSVDAG binary stream.
    voxel_grid: 3D array of size 2^n x 2^n x 2^n, values 0 or 1.
    n: level count (grid size 2^n).
    Returns: binary stream (list of bits).
    """
    # Step 1: Build SVO and SVDAG (Bottom-Up Reduction)
    # This step produces a DAG where nodes are identified by their child structure.
    # We assume a function build_svodag that returns a list of nodes per level.
    # Each node has: id, child_mask (8 bits), children (list of node ids or None), 
    # and frequency (number of parents referencing it).
    svodag_nodes = build_svodag(voxel_grid, n)
    
    # Step 2: Frequency-Based Label Assignment
    # For each level l from n-2 down to 0
    level_labels = {} # Maps level l to a list of (SIZ, VAL) tuples
    
    for l in range(n - 2, -1, -1):
        nodes_at_level = svodag_nodes[l]
        # Identify nodes referenced more than once
        frequent_nodes = [node for node in nodes_at_level if node.frequency > 1]
        
        if not frequent_nodes:
            level_labels[l] = []
            continue
            
        # Sort by frequency descending
        frequent_nodes.sort(key=lambda x: x.frequency, reverse=True)
        
        # Generate labels sorted by length ascending
        # Label 0: SIZ=0, VAL=0 (length 1 bit)
        # Label 1: SIZ=0, VAL=1 (length 1 bit) ... up to SIZ=0, VAL=1
        # Then SIZ=1, VAL=0 (length 2 bits), etc.
        labels = []
        siz = 0
        val = 0
        while len(labels) < len(frequent_nodes):
            # Length of this label is siz + 1
            labels.append((siz, val))
            val += 1
            if val >= (1 << (siz + 1)):
                val = 0
                siz += 1
                if siz > 31:
                    raise ValueError("Label space exhausted")
        
        # Assign labels to nodes
        for i, node in enumerate(frequent_nodes):
            node.label = labels[i]
            
        level_labels[l] = labels

    # Step 3: Encode Binary Stream
    stream = []
    
    # Helper to encode a node's binary representation recursively
    def encode_node(node_id, level):
        node = svodag_nodes[level][node_id]
        
        # Leaf Node (Level n-1)
        if level == n - 1:
            # Encode 8 voxels based on space-filling curve
            voxels = get_voxels_for_octant(node_id, level, n)
            stream.extend(voxels)
            return

        # Inner Node
        # 1. Encode ACHNC
        active_count = count_active_children(node)
        achnc_val = active_count - 1
        stream.extend(int_to_bits(achnc_val, 3))
        
        # 2. Encode CHNM (Header Tags)
        # Determine which children are active
        children = node.children
        active_indices = [i for i, c in enumerate(children) if c is not None]
        
        # Iterate through octants 0 to 7
        for i in range(8):
            child_id = children[i]
            
            # Passive child
            if child_id is None:
                # Only encode '00' if it appears before the last active child
                # Or if it's the only child? No, passive children after last active are omitted.
                # We need to know if this is the last active child.
                if i in active_indices:
                    # This is an active child.
                    # Check if it's the last active one
                    is_last_active = (i == max(active_indices))
                    
                    # Determine if this child is a template or a caller
                    # In the first pass (Depth-First), we encounter the template first.
                    # We need to know if this specific reference is the first one.
                    # This requires tracking state or a two-pass approach.
                    # For simplicity, assume we are building the stream in a way that
                    # templates are encountered before callers.
                    
                    # Check if this node has a label assigned (it is a common subtree)
                    if node_id in get_common_subtree_ids(level):
                        # It is a common subtree.
                        # If this is the first reference (template), encode Label + Node
                        # If this is a subsequent reference, encode Caller
                        # Note: The source implies Depth-First traversal ensures Label comes first.
                        # We need to track if we have already encoded the template for this node_id.
                        if not is_template_encoded(node_id):
                            mark_template_encoded(node_id)
                            stream.extend("01") # HT for Label
                            siz, val = node.label
                            stream.extend(int_to_bits(siz, 5))
                            stream.extend(int_to_bits(val, siz + 1))
                            encode_node(child_id, level + 1)
                        else:
                            stream.extend("10") # HT for Caller
                            siz, val = node.label
                            stream.extend(int_to_bits(siz, 5))
                            stream.extend(int_to_bits(val, siz + 1))
                    else:
                        # Unique node
                        stream.extend("11") # HT for Direct Node
                        encode_node(child_id, level + 1)
                else:
                    # Passive child.
                    # Only encode if it is NOT after the last active child.
                    if i <= max(active_indices) if active_indices else True:
                        stream.extend("00") # HT for Passive
                    # Else: omit
            else:
                # Active child (child_id is not None)
                # This logic is handled inside the 'if i in active_indices' block above
                pass
                
        # Note: The above pseudocode simplifies the "first reference" logic.
        # In practice, a global state or a two-pass construction is needed to determine
        # which reference is the template.
```

**Edge Cases and Pitfalls:**
*   **Passive Child Omission:** Passive children (HT '00') appearing after the last active child are omitted from the CHNM. The ACHNC is required to reconstruct the full 8-child structure or convert to SVDAG [S1].
*   **Label Uniqueness:** Labels are unique only within a level. A label at level $l$ with $(SIZ, VAL)$ is distinct from a label at level $l'$ with the same $(SIZ, VAL)$ [S1].
*   **Traversal Order:** Depth-First traversal is required to ensure that a Label (template) is processed before any Callers referencing it [S1].
*   **Precision:** SIZ is limited to 5 bits, allowing VAL up to 32 bits. This limits the maximum number of unique nodes per level to $2^{32}$, but practically constrained by the grid size [S1].

## Subtopics

### [PSVDAG Label/Caller Encoding with Frequency-Based Compaction](1152-psvdag-label-caller-encoding-with-frequency-based-compaction.md)
This variant focuses on the specific algorithm for assigning variable-length labels based on node reference frequency to minimize the total bit length of the structure. It sets itself apart by optimizing the label-to-node mapping to achieve higher compression, at the cost of increased construction complexity.

## Operating characteristics
*   **Time Complexity:** Construction requires bottom-up reduction and frequency analysis, similar to SVDAG construction, plus label assignment. Traversal is more complex than SVO/SVDAG due to label resolution, requiring a stack or recursion to resolve callers to templates [S2].
*   **Memory Complexity:** Significantly lower than SVDAG and PSVO. By eliminating 32-bit pointers and using variable-length labels, PSVDAG achieves compression ratios up to 5.33-fold compared to standard SVDAGs [S1, S2].
*   **Performance:** Suitable for archiving and streaming due to compact binary representation [S2]. Unsuitable for real-time rendering or quick traversal due to the overhead of resolving labels and callers [S2].
*   **Scaling:** Compression efficiency improves with scene redundancy (common subtrees). The fixed 5-bit SIZ limits the label space per level, but this is sufficient for typical voxel grid sizes [S1].

## Background
PSVDAGs were proposed in 2020 [S2]. They were developed to combine the advantages of PSVOs (space efficiency) and SVDAGs (common subtree merging) [S2]. The structure addresses the high memory cost of pointers in SVDAGs while overcoming the lack of CSM in PSVOs [S2].

## Sources
[S1] PSVDAG: Compact Voxelized Representation of 3D Scenes Using Pointerless Sparse Voxel Directed Acyclic Graphs, https://doi.org/10.31577/cai_2020_3_587, fetched 2026-10-26
[S2] Transforming Hierarchical Data Structures – a PSVDAG–SVDAG Conversion Algorithm, https://doi.org/10.12700/aph.18.8.2021.8.3, fetched 2026-10-30
