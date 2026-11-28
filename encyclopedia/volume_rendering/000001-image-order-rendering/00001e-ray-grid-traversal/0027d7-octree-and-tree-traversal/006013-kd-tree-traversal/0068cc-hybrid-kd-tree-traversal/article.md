# Hybrid Kd Tree Traversal

Also called: JiTTree traversal, Recursive Kd Tree of Bricks

## Overview
Hybrid Kd Tree Traversal is a method for efficiently accessing heterogeneous volume data by organizing distinct brick types (e.g., dense grids, octrees, voxel lists) into a hierarchical Kd tree structure [S1, unofficial]. It solves the problem of sparse or mixed-format volumes where a single uniform data structure would waste memory or require complex type-checking during ray traversal [S1, unofficial]. Unlike its sibling [Adaptive Kd Tree](20471-adaptive-kd-tree.md), which adapts subdivision based on data density, this method groups bricks by their storage format to enable specialized, type-specific access routines at the leaves [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume dataset partitioned into bricks of varying types (Dense, Octree, Voxel List, Empty).
- Output: A Kd tree structure where internal nodes define spatial splits and leaf nodes point to homogeneous brick groups or specific access functions.

**Data Structures:**
The tree consists of `Node` structures. Each node contains:
- `axis`: Integer (0, 1, 2) indicating the splitting dimension (x, y, z) [S1, unofficial].
- `key`: Integer coordinate value of the splitting plane [S1, unofficial].
- `type`: Enum indicating the node's role. Values include `INTERNAL`, `OCTREE`, `VLIST`, `DENSE`, and `EMPTY` [S1, unofficial].
- `leftChild`: Pointer or index to the left child node [S1, unofficial].
- `rightChild`: Pointer or index to the right child node [S1, unofficial] [G].

**Algorithm Steps:**
1. **Construction:** Recursively split the volume space to minimize the entropy of brick types in the resulting sub-regions [S1, unofficial].
2. **Traversal:** For a query position `pos`, traverse from the root:
   - If `node.type == INTERNAL`:
     - Compare `pos[node.axis]` with `node.key`.
     - If `pos[node.axis] < node.key`, recurse on `leftChild`.
     - Else, recurse on `rightChild` [S1, unofficial].
   - If `node.type != INTERNAL`:
     - Route to the specific handler for the leaf type (e.g., `getDense`, `getOctree`, `getVList`) or return `EMPTY` [S1, unofficial].

**Formulas:**
The construction uses entropy minimization. The entropy $E(S)$ of a region $S$ with respect to brick types is:
$$ E(S) = - \sum_{i=1}^{b} p_i \log_2 p_i $$
where $b$ is the number of brick types and $p_i$ is the proportion of bricks with type $i$ in region $S$ [S1, unofficial].

The average entropy $I(S)$ for a split creating sub-regions $S_j$ is:
$$ I(S) = \sum_{j=1}^{2} \frac{v_j}{v} E(S_j) $$
where $v_j$ is the volume of sub-region $S_j$ and $v$ is the total volume of the parent region [S1, unofficial]. The split that minimizes $I(S)$ is selected [S1, unofficial].

**Parameters:**
- Number of brick types ($b$): Typically 4 (Dense, Octree, Voxel List, Empty) in standard implementations [S1, unofficial].

**Pitfalls:**
- **OpenCL Compatibility:** OpenCL does not support `switch` statements. Type routing must be implemented using `if/else` chains, which offer comparable performance to switches in this context [S1, unofficial].
- **Precision:** Splitting planes are defined by integer coordinates. Bricks sharing coordinates must be handled carefully during sorting to ensure balanced splits [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Node structure definition
class Node:
    axis: int       # 0=x, 1=y, 2=z
    key: int        # Splitting coordinate
    type: int       # INTERNAL, OCTREE, VLIST, DENSE, EMPTY
    leftChild: Node # Reference to left child
    rightChild: Node # Reference to right child

# Traversal function
def get(pos: list, node: Node) -> int:
    """
    pos: list of 3 integers [x, y, z]
    node: current tree node
    returns: brick data or EMPTY
    """
    if node.type == INTERNAL:
        # Compare position along the splitting axis
        if pos[node.axis] < node.key:
            return get(pos, node.leftChild)
        else:
            return get(pos, node.rightChild)
    
    # Leaf node type routing
    elif node.type == OCTREE:
        return getOctree(pos, node)
    elif node.type == VLIST:
        return getVList(pos, node)
    elif node.type == DENSE:
        return getDense(pos, node)
    else:
        return EMPTY
```

## Subtopics

### [Kd Tree Unrolling (JiTTree)](20326-kd-tree-unrolling-jittree.md)
JiTTree is a data structure optimization that uses just-in-time (JIT) compilation to specialize tree traversal code for a specific dataset, eliminating type-checking overhead and pointer-chasing by unrolling the tree structure into nested conditionals. It builds on [Hybrid Kd Tree Traversal](20234-hybrid-kd-tree-traversal.md) by grouping homogeneous brick types into leaf nodes and applying partial evaluation at runtime to inline type-specific logic, thereby shifting the performance bottleneck from memory-bound data fetches to instruction-cache-bound code execution.

### Entropy-Based Kd Tree Construction
A kd-tree construction algorithm that splits nodes to minimize the average entropy of brick types in child sub-regions, effectively grouping bricks of the same type together [S1, unofficial].
**Steps:**
1. Sort bricks with respect to x, y, and z coordinates separately [S1, unofficial].
2. For each coordinate, find the split candidate where left and right sub-lists are closest to equally sized [S1, unofficial].
3. Calculate the average entropy $I(S)$ for each possible split using the formula $I(S) = \sum_{j=1}^{2} \frac{v_j}{v} E(S_j)$ [S1, unofficial].
4. Select the split that minimizes $I(S)$ [S1, unofficial].
5. Recursively apply splitting until each kd-tree leaf contains bricks of only one elemental brick type [S1, unofficial].
**Example:** Split at x=192 in a 320-volume region. Left sub-volume (weight 3/8) has entropy 0.5436. Right sub-volume (weight 5/8) has entropy 0.6775. Weighted average entropy $I(S) = 0.627$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Traversal is $O(\log N)$ where $N$ is the number of bricks, assuming a balanced tree [S1, unofficial]. Construction is more expensive due to sorting and entropy calculation.
- **Memory Complexity:** Depends on the tree depth and the number of bricks. The tree structure itself is small compared to the volume data [S1, unofficial].
- **Strengths:** Efficiently handles heterogeneous data; reduces memory overhead for sparse volumes; allows specialized access for each brick type [S1, unofficial].
- **Limitations:** Construction is complex; JIT compilation adds runtime overhead; OpenCL compatibility requires careful coding [S1, unofficial].
- **When to use:** Heterogeneous volumes with mixed brick types (dense, octree, voxel lists) where uniform storage is inefficient [S1, unofficial].
- **When not to use:** Uniform dense volumes where a simple grid or octree suffices [S1, unofficial].

## Background
The method originates from research on adaptive, hybrid data structures for volume datasets on GPUs [S1, unofficial]. It builds on the concept of Kd trees for sparse data representation [S1, unofficial]. The entropy-based construction criterion was introduced by Michalski et al. in the context of optimizing decision trees [S1, unofficial].

## Sources
[S1] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
