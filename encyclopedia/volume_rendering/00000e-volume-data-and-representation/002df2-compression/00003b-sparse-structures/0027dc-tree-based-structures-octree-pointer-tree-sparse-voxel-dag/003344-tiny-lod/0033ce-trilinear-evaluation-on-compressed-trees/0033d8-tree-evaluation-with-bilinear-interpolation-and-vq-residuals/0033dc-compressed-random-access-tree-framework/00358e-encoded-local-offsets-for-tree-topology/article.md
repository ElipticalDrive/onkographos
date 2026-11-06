# Encoded Local Offsets for Tree Topology
Also called: Local offset encoding, Scaled offset encoding

## Overview
Encoded local offsets is a topology compression scheme that replaces absolute pointers in an adaptive tree structure with compact, locally-scaled offsets, enabling random access without pointer indirection [S1, unofficial]. It sets itself apart from prior offset-based methods by defining offsets relative to the current node rather than the start of the data structure, allowing for a unified 1-byte `PointerOrData` layout [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An "autumnal" tree structure where internal nodes contain data and children are either pointers to other nodes or leaf data values [S1, unofficial].
- Output: A memory layout where nodes are packed in preorder, and child references are replaced by 7-bit encoded local offsets [S1, unofficial].

**Data Structures:**
The core structure is `PointerOrData`, which is exactly 1 byte (8 bits) [S1, unofficial]:
- Bit 0 (1 bit): `leafchild` flag. If set, the remaining 7 bits represent leaf data. If clear, the remaining 7 bits represent a local offset to a child node [S1, unofficial].
- Bits 1-7 (7 bits): Either the encoded leaf data value or the encoded local offset value $x \in [0, 127]$ [S1, unofficial].
- Data values are encoded into 7 bits to fit within this structure [S1, unofficial].

**Algorithm:**
The packing process proceeds from fine levels to coarse levels (Level $L-1$ down to 0) [S1, unofficial]:

1. **Initialize:** Start with the autumnal tree structure [S1, unofficial].
2. **Iterate Levels:** For each level $l$ from $L-1$ down to 0:
   a. Identify all parent nodes at level $l$ that have already been placed in memory.
   b. Identify the subtrees rooted at the children of these parents. These subtrees have already been packed in previous iterations (for finer levels) or are leaf nodes [S1, unofficial].
   c. **Concatenate:** Append the memory blocks of the children's subtrees immediately after their respective parent nodes in the global memory buffer [S1, unofficial].
   d. **Padding:** If the distance (offset) from the parent to the start of a child's subtree is not an exact multiple of the scaling factor $s$ for that level, insert padding bytes between the parent and the child block to align the offset [S1, unofficial].
3. **Scaling Factor Selection:** For each level, determine the integer scaling factor $s$ [S1, unofficial]:
   a. Calculate $y_{max}$, the largest required offset from a parent to its last child at this level [S1, unofficial].
   b. Search for $s$ in the range $[\lceil y_{max} / 127 \rceil, y_{max}]$ [S1, unofficial].
   c. Select the $s$ that minimizes padding waste while ensuring all offsets $y$ can be represented as $y = s \cdot x$ for some integer $x \in [0, 127]$ [S1, unofficial].
   d. At fine levels, $s=1$ is typically used, making the encoding wasteless [S1, unofficial].

**Formula:**
The local offset $y$ (in bytes) is encoded into a 7-bit value $x$ using a per-level scaling parameter $s$:
$$ y = s \cdot x $$
Where:
- $y$ is the local offset in bytes from the current node to the child node [S1, unofficial].
- $x$ is the 7-bit encoded value, $x \in [0, 127]$ [S1, unofficial].
- $s$ is the integer scaling parameter for the current level [S1, unofficial].

**Parameters:**
- **Scaling factor $s$:** An integer per-level parameter [S1, unofficial].
  - Fine levels: $s=1$ [S1, unofficial].
  - Coarse levels: $s>1$ to handle larger offsets [S1, unofficial].
  - Selection: Chosen from $[\lceil y_{max} / 127 \rceil, y_{max}]$ to minimize padding [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Coarse Level Padding:** At coarse levels, offsets are large, leading to significant padding waste if $s$ is not chosen carefully [S1, unofficial].
- **Mitigation:** To mitigate padding and other drawbacks (runtime cost, ineffective VQ), the coarsest tree levels (0-4) are collapsed into a non-adaptive mipmap pyramid [S1, unofficial]. A 172-entry indirection table is stored at the finest pyramid level pointing to clipped subtrees [S1, unofficial].
- **Offset Definition:** Offsets are local (0 refers to memory just after the current node), not absolute from the start of the tree [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global memory buffer
memory = []

# Tree structure: nodes are objects with 'children' list and 'data'
# 'children' contains either 'leaf_data' or 'child_node' references
# 'data' is the value stored in internal nodes

def pack_tree(root_node, levels):
    # levels is a list of lists, where levels[l] contains all nodes at level l
    # Process from finest (L-1) to coarsest (0)
    for l in range(len(levels) - 1, -1, -1):
        current_level_nodes = levels[l]
        
        # Determine scaling factor s for this level
        # First, we need to know the max offset y_max
        # This requires a preliminary pass or dynamic calculation
        # For simplicity, assume we calculate y_max based on subtree sizes
        y_max = calculate_max_offset(current_level_nodes, memory)
        
        # Search for optimal s
        min_padding = infinity
        best_s = 1
        start_s = ceil(y_max / 127)
        
        for s in range(start_s, y_max + 1):
            padding = 0
            for node in current_level_nodes:
                for child in node.children:
                    # Calculate required offset y
                    # This depends on where the child's subtree is currently placed
                    # In the iterative process, children are already placed if they are leaves
                    # or if their subtrees were packed in finer levels
                    child_start_addr = get_address_of(child)
                    parent_end_addr = get_address_after(node)
                    y = child_start_addr - parent_end_addr
                    
                    # Check if y is representable
                    if y % s != 0:
                        padding += (s - (y % s)) % s # Add padding to align to multiple of s
                    # If y is not a multiple of s, we must pad
                    # Actually, the algorithm says: "leave some padding space... if desired offsets cannot be encoded exactly"
                    # So we adjust the placement to make y a multiple of s
            
            if padding < min_padding:
                min_padding = padding
                best_s = s
        
        s = best_s
        
        # Now perform the packing for this level
        for node in current_level_nodes:
            # Node is already in memory from previous steps (or root)
            # Place children after this node
            for child in node.children:
                if child is a leaf:
                    # Encode leaf data into 7 bits
                    encoded_data = encode_data(child.data)
                    # Append 1 byte: leafchild=1, data=encoded_data
                    memory.append(0x80 | encoded_data)
                else:
                    # Child is a subtree root, already packed in memory
                    # Calculate current offset
                    parent_end = len(memory)
                    child_start = get_address_of(child)
                    y = child_start - parent_end
                    
                    # Calculate required padding to make y a multiple of s
                    remainder = y % s
                    if remainder != 0:
                        padding = s - remainder
                        memory.extend([0] * padding) # Pad with zeros or specific marker
                    
                    # Now y is a multiple of s
                    y = len(memory) - parent_end
                    x = y // s
                    
                    # Encode offset into 7 bits
                    # leafchild=0, offset=x
                    encoded_offset = x & 0x7F
                    memory.append(encoded_offset)
                    
                    # Note: The child's subtree is already in memory, so we don't append it again
                    # We just record the offset to it

def calculate_max_offset(nodes, memory):
    # Helper to estimate max offset for scaling factor selection
    # This is a simplification; in practice, it might require a two-pass approach
    max_y = 0
    for node in nodes:
        for child in node.children:
            if child is not a leaf:
                # Estimate distance
                # This is complex because children are interleaved
                # The source implies an exhaustive search, so we assume y_max is known or estimated
                pass
    return max_y

def encode_data(data):
    # Encode data value into 7 bits
    # Implementation depends on data type
    return data & 0x7F

def get_address_of(node):
    # Returns the index in 'memory' where the node's PointerOrData byte is stored
    # This requires tracking node positions during packing
    pass

def get_address_after(node):
    # Returns the index in 'memory' immediately after the node's PointerOrData byte
    return get_address_of(node) + 1
```

## Subtopics

### Hierarchical Delta Encoding and Quantization
A sibling method that focuses on compressing the data values using hierarchical prediction and quantization, rather than the topology encoding described here. It complements the offset encoding by handling the data residuals [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The packing process is $O(N)$ where $N$ is the number of nodes, as each node is visited once. The search for the optimal scaling factor $s$ adds a constant factor per level, depending on the range of $s$ [S1, unofficial].
- **Memory Complexity:** The topology is stored at 0.36 bytes/node for the example quadtree, which is significantly smaller than storing 32-bit pointers [S1, unofficial]. The total memory includes the data values, which are also encoded into 7 bits [S1, unofficial].
- **Performance:** The method reduces memory size by eliminating pointer indirection and padding. The forested mipmap structure further reduces memory by collapsing coarse levels [S1, unofficial].
- **Strengths:** Compact representation, random access capability, no pointer indirection overhead [S1, unofficial].
- **Limitations:** Padding waste at coarse levels if not mitigated by the mipmap structure [S1, unofficial].

## Background
The method was introduced in "Compressed Random-Access Trees for Spatially Coherent Data" [S1, unofficial]. It builds on the concept of replacing absolute pointers with offsets, as previously explored by Hunter and Willis [HW91], but improves upon it by using local offsets and a 1-byte structure [S1, unofficial]. The technique is part of a broader framework for compressing adaptive multiresolution hierarchies [S1, unofficial].

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
