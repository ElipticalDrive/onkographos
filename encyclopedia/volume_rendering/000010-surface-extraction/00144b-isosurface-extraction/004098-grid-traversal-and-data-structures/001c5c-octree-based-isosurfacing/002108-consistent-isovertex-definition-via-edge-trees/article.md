# Consistent Isovertex Definition via Edge-Trees

Also called: Edge-Tree Isovertex Definition, Edge-Tree Traversal for Isovertex

## Overview
This method defines the position of an isovertex on an octree edge consistently by traversing an "edge-tree" to a unique leaf node with value 1, ensuring watertightness in isosurface extraction [S1, S2, unofficial]. It solves the problem of ambiguous isovertex positions when a coarse edge contains multiple sign changes, which causes standard polygonization methods like [Blo88] to fail [S2, unofficial]. Unlike methods that refine the tree or modify vertex values, this approach uses the existing edge-tree structure to resolve ambiguity, contributing to closed isopolyline loops and shared polygon edges [S2, unofficial]. It differs from the minimal edge definition in [JLSW02] only when there is more than one isovalue-crossing along an edge [S1, unofficial].

## How it works
**Inputs:**
- An octree representing the scalar field [G].
- An edge-tree structure associated with the octree edges [S1, S2, unofficial].
- A scalar isovalue [G].

**Outputs:**
- A set of isovertices, each associated with an octree edge [S1, S2, unofficial].

**Data Structures:**
- **Edge-Tree Node:** Represents an octree edge or a sub-segment of it [S1, S2, unofficial].
    - `value`: Binary integer (0 or 1). 1 indicates an isovalue crossing (sign change) exists in the segment represented by this node. 0 indicates no crossing [S1, S2, unofficial].
    - `children`: List of child nodes (for interior nodes). Leaf nodes have no children [G].
    - `geometry`: Geometric representation of the edge segment (start point, end point) for interpolation [G].

**Algorithm:**

1. **Identify Crossing Edges:** For every edge $e$ in the octree, determine if it crosses the isovalue by checking if the scalar values at the endpoints have different signs relative to the isovalue [G].
2. **Compute Edge-Tree Values:** For every interior node in the edge-tree, compute its `value` based on its children. The binary value of an interior node is the sum modulo 2 of its children's values [S1, S2, unofficial]:
   $$
   \text{value}(\text{parent}) = (\text{value}(\text{child}_1) + \text{value}(\text{child}_2)) \pmod 2
   $$
   This property guarantees that if an edge $e$ has `value = 1`, it must have a unique child that also has `value = 1` [S1, S2, unofficial].
3. **Traverse to Leaf:** For each edge $e$ with `value = 1`:
   - Start at the root node of the edge-tree corresponding to $e$ [S1, S2, unofficial].
   - Recursively traverse down the unique child that has `value = 1` [S1, S2, unofficial].
   - Continue until a leaf node $\tilde{e}$ is reached [S1, S2, unofficial].
   - The path $e \supseteq \dots \supseteq \tilde{e}$ consists entirely of nodes with `value = 1` [S1, S2, unofficial].
4. **Define Isovertex Position:** Assign the isovertex position $p$ to be the position defined by the leaf node $\tilde{e}$ [S1, S2, unofficial]. Denote this assignment as $p \in e$.
   - The position $p$ is interpolated within the segment represented by $\tilde{e}$ based on the scalar values at the endpoints of $\tilde{e}$ [G].
   - Note: $p \in e$ and $e' \supseteq e$ does not imply $p \in e'$ [S1, unofficial]. The isovertex is tied to the specific leaf $\tilde{e}$, not the coarse edge $e$ [S1, unofficial].
5. **Handle Multiple Crossings:** If an edge has multiple sign changes, standard polygonization fails [S2, unofficial]. This method handles it by assigning the isovertex to the finest leaf node in the edge-tree that still indicates a crossing, effectively resolving ambiguity [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_edge_tree_values(node):
    """Recursively computes the binary value of an edge-tree node based on its children [S1, S2, unofficial]."""
    if node.is_leaf():
        return node.value
    child_values = [compute_edge_tree_values(child) for child in node.children]
    node.value = sum(child_values) % 2
    return node.value

def find_isovertex(edge_root):
    """Traverses the edge-tree to find the leaf node defining the isovertex [S1, S2, unofficial]."""
    if edge_root.is_leaf():
        return edge_root
    for child in edge_root.children:
        if child.value == 1:
            return find_isovertex(child)
    raise ValueError("No child with value 1 found")

def interpolate_position(leaf_node, isovalue):
    """Linear interpolation within the leaf segment based on endpoint scalar values [G]."""
    t = (isovalue - leaf_node.scalar_start) / (leaf_node.scalar_end - leaf_node.scalar_start)
    return leaf_node.start_point + t * (leaf_node.end_point - leaf_node.start_point)

def define_isovertices(octree_edges, isovalue):
    """Defines isovertices for all edges in the octree [G]."""
    isovertices = {}
    for edge in octree_edges:
        if edge.has_sign_change():
            edge.edge_tree_root.value = 1
            compute_edge_tree_values(edge.edge_tree_root)
    for edge in octree_edges:
        if edge.edge_tree_root.value == 1:
            leaf_node = find_isovertex(edge.edge_tree_root)
            p = interpolate_position(leaf_node, isovalue)
            isovertices[edge] = p
    return isovertices
```

**Parameters:**
- **Isovalue:** The scalar threshold for extraction [G]. Typical values are not given in the sources [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Multiple Sign Changes:** Standard polygonization fails if an edge has multiple sign changes [S2, unofficial]. This method handles it by assigning the isovertex to the finest leaf node in the edge-tree that still indicates a crossing [S2, unofficial].
- **Notation Ambiguity:** The notation $p \in e$ does not imply $p \in e'$ for an ancestor edge $e' \supseteq e$ [S1, unofficial]. The isovertex is tied to the specific leaf $\tilde{e}$ in the edge-tree, not the coarse edge $e$ itself [S1, unofficial].
- **Precision:** Interpolation within the leaf segment may introduce precision errors if the leaf segment is very small or if the scalar field values are close to the isovalue [G].

## Subtopics

### [Twin Isovertex Pairing for Isopolyline Closure](4606-twin-isovertex-pairing-for-isopolyline-closure.md)
Twin Isovertex Pairing is a post-processing step in octree-based isosurface extraction that closes open isopolylines into closed isopolygons by identifying and connecting "twin" isovertices. It solves the problem of watertightness in meshes extracted from arbitrary octrees where a single edge may contain multiple sign changes, resulting in isovertices with valence one that cannot be paired within the local leaf node. Unlike methods that refine the tree or modify vertex values, this method uses the existing edge-tree structure to pair vertices, ensuring the resulting mesh is watertight without altering the underlying scalar field representation.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, S2, unofficial].
- **Memory Complexity:** Not given in the sources [S1, S2, unofficial].
- **Strengths and Limitations:** Not given in the sources [S1, S2, unofficial].
- **When to Use / When Not to Use:** Not given in the sources [S1, S2, unofficial].

## Background
This method was introduced in the paper "Unconstrained Isosurface Extraction on Arbitrary Octrees" [S1, S2, unofficial]. It addresses the challenges of isosurface extraction on arbitrary octrees, where traditional methods like Marching Cubes may produce cracks or non-watertight meshes due to inconsistent isovertex definitions [S1, unofficial]. The method builds on the properties of edge-trees to ensure consistency [S1, S2, unofficial].

## Sources
[S1] Unconstrained isosurface extraction on arbitrary octrees, https://doi.org/10.5555/1281991.1282009, fetched 2026-10-27 (unofficial)
[S2] Unconstrained Isosurface Extraction on Arbitrary Octrees, https://hhoppe.com/unconstrainediso.pdf, fetched 2026-10-30 (unofficial)
