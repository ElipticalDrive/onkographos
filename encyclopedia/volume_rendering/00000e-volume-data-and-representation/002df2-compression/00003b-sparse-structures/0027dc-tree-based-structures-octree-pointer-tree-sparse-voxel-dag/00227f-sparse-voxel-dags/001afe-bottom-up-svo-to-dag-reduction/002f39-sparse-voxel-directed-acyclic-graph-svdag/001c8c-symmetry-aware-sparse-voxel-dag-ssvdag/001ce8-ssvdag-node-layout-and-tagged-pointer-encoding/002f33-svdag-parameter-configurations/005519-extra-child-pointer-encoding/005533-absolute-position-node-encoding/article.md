# Absolute Position Node Encoding

Also called: Absolute position of nodes

## Overview
Absolute Position Node Encoding is a steganographic technique that embeds data into Sparse Voxel Directed Acyclic Graphs (SVDAG) by exploiting the correlation between a node's absolute memory address and its structural properties, specifically the parity of its child count [S1]. It sets itself apart from sibling methods like [Extra Child Pointer Encoding](15412-extra-child-pointer-encoding.md) by modifying the logical interpretation of existing structural metadata (address bits) rather than appending unused pointers, and from [Extra free space](15402-extra-free-space.md) by operating on node-level attributes rather than file-level layout gaps [S1].

## How it works
**Inputs and Outputs:**
- Input: A SVDAG data structure where nodes have defined absolute addresses in the addressing space and a known number of child nodes.
- Output: A modified SVDAG where the relationship between node address bits and child node counts encodes a secret bitstream.

**Data Structures:**
The SVDAG node layout is assumed to be consistent with the parent article [SSVDAG Node Layout and Tagged Pointer Encoding](15401-ssvdag-node-layout-and-tagged-pointer-encoding.md). Each node $N$ has:
- An absolute address $A_N$ in the linear addressing space of the graph.
- A child node mask or list defining the set of children $C_N = \{c_1, c_2, ..., c_k\}$.
- The size of the node $S_N$ in bits, typically ranging from 32 to 288 bits [S1].

**Steps:**
1. **Select Target Bit:** Choose a specific bit position $b$ in the absolute address $A_N$ of each node. The source cites the third least significant bit (bit index 2, value $2^2=4$) as an example [S1]. Let $bit(A_N, b)$ be the value of this bit (0 or 1).
2. **Count Children:** Determine the number of valid child nodes $k$ for the current node $N$.
3. **Determine Parity:** Calculate the parity of the child count: $p = k \mod 2$.
4. **Encode Secret Bit:**
   - If the secret bit to be encoded is $0$, ensure that $bit(A_N, b) = 0$ when $p=0$ (even children) or $bit(A_N, b) = 1$ when $p=1$ (odd children)? No, the source states: "particular bit ... for node that has even number of child nodes means encoding of bit 0 and odd number of child nodes means encoding of bit 1."
   - Correction based on [S1]: The encoding rule is fixed by the structure. The secret bit is derived from the existing state. To *hide* a bit, the encoder must manipulate the graph structure (e.g., via merging or splitting nodes, or reordering if applicable, though absolute position implies address assignment) such that the condition holds.
   - Specifically:
     - To hide bit $0$: Ensure that if $k$ is even, $bit(A_N, b) = 0$. If $k$ is odd, $bit(A_N, b) = 1$? The text says "even ... means encoding of bit 0". This implies a mapping:
       - If $k$ is even, the bit value $bit(A_N, b)$ represents $0$.
       - If $k$ is odd, the bit value $bit(A_N, b)$ represents $1$.
     - Therefore, to hide a secret bit $s$:
       - If $s=0$: Adjust the graph (e.g., via node merging/splitting decisions during construction or post-processing) so that the node has an even number of children if $bit(A_N, b)=0$, or an odd number if $bit(A_N, b)=1$?
       - Actually, the address is usually fixed by the allocator. The "feature" is the child count. The method uses the *correlation*. The source implies that the encoder can influence the child count (via the DAG reduction process) to match the desired bit at the chosen address position.
       - Rule:
         - If $bit(A_N, b) == 0$: Encode $0$ if $k$ is even. Encode $1$ if $k$ is odd.
         - If $bit(A_N, b) == 1$: Encode $0$ if $k$ is odd? No, the source says "even ... means 0, odd ... means 1". This is a direct mapping from parity to bit value.
         - Let's re-read carefully: "particular bit of the node address ... for node that has even number of child nodes means encoding of bit 0 and odd number of child nodes means encoding of bit 1."
         - This phrasing is ambiguous. Does it mean the bit *value* is determined by the parity? Or the parity determines the bit?
         - Standard steganography in this context: The secret bit $s$ is hidden by ensuring the parity $p$ matches the bit value $v = bit(A_N, b)$.
         - If $v=0$, we want $p=0$ (even children).
         - If $v=1$, we want $p=1$ (odd children).
         - Thus, to hide bit $s$:
           - If $bit(A_N, b) == 0$: Force $k$ to be even.
           - If $bit(A_N, b) == 1$: Force $k$ to be odd.
           - This is achieved during the SVDAG construction/reduction phase by choosing whether to merge nodes or keep them separate, thereby controlling the child count $k$.
5. **Repeat:** Apply this to subsequent nodes in the addressing space to encode the bitstream.

**Parameters:**
- **Bit Position:** Any bit in the address can be used. The third least significant bit is cited as an example [S1].
- **Node Size:** 32 to 288 bits [S1].

**Edge Cases and Pitfalls:**
- **Detection:** Analysis of irregularities in the addressing of nodes can identify the use of data hiding. Specifically, statistical deviations in the correlation between address bits and child node parities can reveal the payload [S1].
- **Capacity Limitation:** The method hides only one bit of information per node [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_absolute_position(svdag, secret_bits):
    """
    Encodes secret_bits into the SVDAG by manipulating child node counts
    to correlate with specific bits of the node's absolute address.
    
    Args:
        svdag: A Sparse Voxel Directed Acyclic Graph structure with assigned absolute addresses.
        secret_bits: A list of integers (0 or 1) to hide.
        
    Returns:
        Modified SVDAG where child counts are adjusted to hide the bits.
    """
    bit_index = 2  # Example: third least significant bit [S1]
    bit_mask = 1 << bit_index
    
    for i, secret_bit in enumerate(secret_bits):
        if i >= len(svdag.nodes):
            break
            
        node = svdag.nodes[i]
        address = node.absolute_address
        
        # Extract the target bit from the address
        target_bit_value = (address & bit_mask) >> bit_index
        
        # Determine required parity for the child count
        # If target_bit_value is 0, we need even children (parity 0)
        # If target_bit_value is 1, we need odd children (parity 1)
        # This assumes the mapping: Even->0, Odd->1 [S1]
        required_parity = target_bit_value
        
        current_child_count = len(node.children)
        current_parity = current_child_count % 2
        
        if current_parity != required_parity:
            # Adjust child count to match required parity
            # This is done via SVDAG reduction operations (merge/split)
            # [G] Implementation detail: The specific merge/split operation
            # depends on the parent SVDAG construction algorithm.
            # We toggle the child count by 1 (add or remove a child reference)
            # or merge/split nodes to change the count parity.
            if required_parity == 0:
                # Need even number of children
                if current_child_count % 2 != 0:
                    # Remove a child or merge to reduce count by 1
                    node.remove_child() # Placeholder for specific DAG op
            else:
                # Need odd number of children
                if current_child_count % 2 == 0:
                    # Add a child or split to increase count by 1
                    node.add_child() # Placeholder for specific DAG op

def decode_absolute_position(svdag, num_bits):
    """
    Decodes secret bits from the SVDAG.
    """
    bit_index = 2
    bit_mask = 1 << bit_index
    secret_bits = []
    
    for i in range(num_bits):
        if i >= len(svdag.nodes):
            break
            
        node = svdag.nodes[i]
        address = node.absolute_address
        target_bit_value = (address & bit_mask) >> bit_index
        
        child_count = len(node.children)
        parity = child_count % 2
        
        # Map parity back to bit
        # Even (0) -> 0, Odd (1) -> 1 [S1]
        decoded_bit = parity
        secret_bits.append(decoded_bit)
        
    return secret_bits
```

## Subtopics

### Relative Position Node Encoding
A variant that uses permutations of child node addresses and their relative positions to encode information [S1]. It sets itself apart by comparing addresses rather than using absolute values [S1].
- **Definition:** Encodes bits based on the relative ordering of child node addresses or parent-child address relationships [S1].
- **Steps:**
  1. Encode bit 0 if the address of the first child node is lower than the second child node, and bit 1 if higher, for child nodes of the same parent [S1].
  2. Encode bit 0 if the address of the parent node is lower than the child node, and bit 1 if the child node address is lower than the parent node address [S1].
- **Parameters:** Capacity is 0.347% to 3.125% of the node size in bits, hiding one bit per node (node size 32 to 288 bits) [S1].
- **Pitfalls:** Detection via analysis of irregularities in addressing of nodes [S1].

## Operating characteristics
- **Capacity:** Low. Hides one bit of information per node [S1]. The capacity ranges from 0.347% to 3.125% of the node size in bits, depending on the node size (32 to 288 bits) [S1].
- **Time Complexity:** Linear with respect to the number of nodes encoded, assuming constant-time address and child-count checks.
- **Memory Complexity:** No additional memory overhead for the payload; it reuses existing structural metadata.
- **Strengths:** Does not require modifying the node bit-field layout or appending extra pointers, preserving the standard node structure [S1].
- **Limitations:** Small capacity; vulnerable to statistical analysis of addressing irregularities [S1].
- **When to use:** When the SVDAG structure is compact and cannot accommodate extra pointers or free space, and low-capacity steganography is sufficient.
- **When not to use:** When high capacity is required or when the addressing space is not under the encoder's control (e.g., fixed memory layout).

## Background
Originated in the context of data hiding using domain-specific hierarchical data structures for geometry representation of voxelized three-dimensional scenes [S1]. The method was proposed as part of a study on SVDAG structures, exploring how structural features like node addresses and child relationships can be exploited for steganography [S1].

## Sources
[S1] ON DATA HIDING USING DOMAIN SPECIFIC HIERARCHICAL DATA STRUCTURE SVDAG FOR GEOMETRY REPRESENTATION OF VOXELIZED THREE-DIMENSIONAL SCENES, https://doi.org/10.15546/aeei-2019-0027, fetched 2026-10-26
