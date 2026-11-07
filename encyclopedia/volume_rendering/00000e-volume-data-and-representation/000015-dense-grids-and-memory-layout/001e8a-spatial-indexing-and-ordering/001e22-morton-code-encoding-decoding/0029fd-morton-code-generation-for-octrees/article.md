# Morton Code Generation for Octrees

Also called: Morton key construction, Z-order curve encoding

## Overview
This method generates unique hierarchical spatial keys for octree cells by recursively concatenating binary octant indices to a parent key. It solves the problem of identifying cells both spatially and hierarchically within a dense or sparse octree structure, enabling efficient neighbor and parent lookups [S1, unofficial]. Unlike standard Morton encoding which interleaves global coordinates, this approach builds keys based on the tree's traversal path, where each level adds exactly three bits to the key length [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The octree structure (or traversal path) and the octant index (0–7) for each child node relative to its parent.
- Output: An integer key (Morton code) for each node in the octree.

**Data Structures:**
- Keys are stored as integers. The bit-length of the key corresponds to the depth of the node in the octree.
- Each octant index is represented by a 3-bit binary value.

**Steps:**
1. **Initialize Root:** Set the key of the root node to the integer $1$ [S1, unofficial].
2. **Recursive Construction:** For any child node, compute its key by concatenating the parent's key with the 3-bit binary representation of the child's octant index [S1, unofficial].
3. **Bit Expansion:** Each level of the octree adds exactly 3 bits to the key length relative to the parent level [S1, unofficial].

**Formulas:**
Let $k_{parent}$ be the key of the parent node and $o$ be the octant index of the child ($0 \le o \le 7$).
The child's key $k_{child}$ is:
$$ k_{child} = k_{parent} \cdot 2^3 + o $$
Or equivalently, using bitwise operations:
$$ k_{child} = (k_{parent} \ll 3) \ | \ o $$

To find the parent key $k_{parent}$ from a child key $k_{child}$:
$$ k_{parent} = k_{child} \gg 3 $$

**Parameters:**
- Root Key: The initial key for the octree root is the integer $1$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_morton_key(parent_key, octant_index):
    """
    Generates the Morton key for a child node given its parent's key and octant index.
    
    Args:
        parent_key (int): The Morton key of the parent node.
        octant_index (int): The octant index of the child (0-7).
        
    Returns:
        int: The Morton key of the child node.
    """
    # Concatenate parent key with 3-bit octant index
    # This is equivalent to shifting parent_key left by 3 bits and OR-ing with octant_index
    child_key = (parent_key << 3) | octant_index
    return child_key

def get_parent_key(child_key):
    """
    Retrieves the parent key from a child's Morton key.
    
    Args:
        child_key (int): The Morton key of the child node.
        
    Returns:
        int: The Morton key of the parent node.
    """
    # Shift right by 3 bits to remove the last 3 bits (octant index)
    parent_key = child_key >> 3
    return parent_key

# Example usage:
# Root key is 1
root_key = 1
# Child in octant 3 (binary 011)
child_key = generate_morton_key(root_key, 3)
# child_key should be (1 << 3) | 3 = 8 | 3 = 11 (binary 1011)
# Note: The source example uses key 97 (binary 1 100 001) and octant 3 (011)
# resulting in 1 100 001 011. This implies the root might not be 1 in that specific example,
# or the example key 97 is not the root. The algorithm remains: shift left 3, add octant.
```

**Edge Cases and Pitfalls:**
- **Key Size:** Keys grow linearly with depth. For deep octrees, 64-bit integers may overflow. Not given in the sources.
- **Root Definition:** The root key is explicitly defined as $1$. If the root is defined differently, the base case must be adjusted [S1, unofficial].

## Subtopics

### [Edge-Octree Construction for Meta-Cell Adjacency](4701-edge-octree-construction-for-meta-cell-adjacency.md)
Constructs a hierarchical adjacency structure to resolve topological disconnections between cells in an adaptive octree. It modifies spatial keys to embed meta-cell identity, creating a unified hierarchy for boundary processing. Choose this when dealing with multi-resolution meshes where cells from different meta-cells share edges.

### [Octree Construction via Morton Codes](9758-octree-construction-via-morton-codes.md)
Constructs a sparse octree from a list of active leaf nodes sorted by Morton code in a bottom-up manner. It efficiently links parent-child relationships and establishes x-neighbor adjacency. Choose this when building an octree from a sorted list of leaves without global sorting or complex traversal.

## Operating characteristics
- **Time Complexity:** Key generation is $O(1)$ per node if the parent key is known. Parent lookup is $O(1)$ via bit shift.
- **Memory Complexity:** Keys are stored as integers. The size of the key is proportional to the depth of the tree. Not given in the sources.
- **Strengths:** Simple implementation, efficient parent/child navigation, preserves spatial locality for adjacent nodes at the same level.
- **Limitations:** Keys do not directly encode global coordinates, making spatial queries (e.g., "find all nodes in region X") require tree traversal or additional indexing. Not given in the sources.

## Background
This method is based on Morton coding [Morton, 1966], adapted for octree hierarchical structures. It is used in volume rendering and isosurface extraction to identify cells spatially and hierarchically [S1, unofficial]. The specific recursive construction described here is detailed in Lewiner et al. [2010] for extracting isosurfaces from massive binary domes [S1, unofficial].

## Sources
[S1] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
