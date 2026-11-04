# Morton Code Generation for Octrees
Also called: Morton key construction, Z-order curve encoding

## Overview
This method generates unique integer keys for octree cells by recursively concatenating octant indices to a parent key, enabling efficient hierarchical spatial indexing and neighbor lookups [S1, unofficial]. It differs from its parent, [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md), which uses bit-interleaving of Cartesian coordinates, by constructing keys via path-based concatenation during tree traversal [S1, unofficial].

## How it works
**Inputs:** Octree structure (nodes with children pointers and octant indices); Root node.
**Outputs:** Dictionary or array mapping each node to its Morton key (integer).

**Data Structures:**
- `Key`: Unsigned integer.
- `OctantIndex`: 3-bit integer (0-7) representing the child's position relative to the parent.

**Steps:**
1. Initialize the root node's key to 1 [S1, unofficial].
2. For each child node of a parent with key $K_{parent}$:
   a. Retrieve the child's octant index $O$ (0-7).
   b. Compute the child's key $K_{child}$ by shifting the parent key left by 3 bits and adding the octant index:
      $$ K_{child} = (K_{parent} \ll 3) + O $$
   c. This effectively concatenates the 3-bit binary representation of $O$ to the binary representation of $K_{parent}$ [S1, unofficial].
3. Recursively apply Step 2 to all children until leaf nodes are reached.

**Parameters:**
- Root Key: 1 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Key Size:** Each level adds 3 bits. For deep trees, keys may exceed standard integer widths (e.g., 64-bit). Use arbitrary-precision integers if depth > 21 [G].
- **Parent Lookup:** To find the parent key $K_{parent}$ from a child key $K_{child}$, perform a right shift by 3 bits:
  $$ K_{parent} = K_{child} \gg 3 $$
  This is valid because the last 3 bits always represent the octant index [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_morton_keys(node, current_key):
    """
    Recursively assigns Morton keys to an octree node and its children.
    
    Args:
        node: The current octree node object.
        current_key: The Morton key of the current node (integer).
    """
    # Assign the computed key to the current node
    node.key = current_key
    
    # If the node has children, process them
    if node.has_children:
        for octant_index in range(8):
            child = node.children[octant_index]
            if child is not None:
                # Compute child key: shift parent key left by 3 bits and add octant index
                child_key = (current_key << 3) + octant_index
                generate_morton_keys(child, child_key)
```

## Subtopics

### [Edge-Octree Construction for Meta-Cell Adjacency](4701-edge-octree-construction-for-meta-cell-adjacency.md)
Constructs a hierarchical adjacency structure by modifying Morton keys to embed meta-cell identity, resolving topological disconnections between cells in adaptive octrees.

### [Meta-Cell Edge Octree Connection](4758-meta-cell-edge-octree-connection.md)
Focuses on final mesh connectivity resolution using the adjacency graph built by its sibling.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of nodes, as each node is visited once with $O(1)$ key computation [G].
- **Memory Complexity:** $O(N)$ to store keys for all nodes [G].
- **Strengths:** Simple implementation; efficient parent/child navigation via bit shifts; preserves spatial locality similar to Z-order curves [S1, unofficial].
- **Limitations:** Keys grow linearly with depth, potentially requiring large integer types; not directly comparable to coordinate-based Morton codes without conversion [G].
- **When to use:** When traversing an already constructed octree and hierarchical path-based indexing is preferred over coordinate-based encoding [S1, unofficial].

## Background
Originated from the need to identify cells spatially and hierarchically in octree structures for isosurface extraction [S1, unofficial]. The method is described in the context of massive binary volume processing, citing Morton (1966) for the general concept of Z-order curves, but adapts it to recursive key concatenation rather than coordinate interleaving [S1, unofficial].

## Sources
[S1] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
