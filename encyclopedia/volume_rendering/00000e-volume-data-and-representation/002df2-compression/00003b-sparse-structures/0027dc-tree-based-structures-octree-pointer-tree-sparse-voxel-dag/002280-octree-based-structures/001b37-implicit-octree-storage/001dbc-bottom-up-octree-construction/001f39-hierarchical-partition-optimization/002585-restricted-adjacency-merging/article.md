# Restricted Adjacency Merging

Also called: Brother/Nephew Merging

## Overview
Restricted Adjacency Merging is a strategy for merging extracted isosurface components from adjacent spatial blocks in sparse volume rendering [S1]. It solves the problem of inefficiently merging blocks that do not share a true geometric boundary by restricting merges to only those blocks (brothers or nephews) that are adjacent in space [S1]. It sets itself apart from unrestricted merging by enforcing an ancestry-based type check to ensure a common boundary exists, preventing the merging of non-adjacent components [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of extracted isosurface components from leaf nodes of an octree, where each component consists of triangles [S1].
*   **Output:** A set of merged, simplified isosurface components ready for dumping to disk [S1].

**Data Structures:**
*   **Node Type:** An enum or integer representing the position of a node relative to its father's split plane: `left`, `right`, `up`, `down`, `front`, `back` [S1].
*   **Ancestry Set ($A_{p,n}$):** A set of nodes containing node $n$ and its recursive fathers up to root $p$ [S1].

**Definitions:**
1.  **Opposite Operator:** Defines the spatial relationship between siblings [S1].
    $$
    \begin{aligned}
    \text{left} &= \text{right}, & \text{right} &= \text{left} \\
    \text{up} &= \text{down}, & \text{down} &= \text{up} \\
    \text{front} &= \text{back}, & \text{back} &= \text{front}
    \end{aligned}
    $$
2.  **Ancestry ($A_{p,n}$):** The set of nodes on the path from $n$ to $p$ [S1]. Defined recursively:
    $$
    A_{p,n} = \begin{cases} 
    \{n\} & \text{if } n = p \\
    \{n\} \cup A_{p,f_n} & \text{else}
    \end{cases}
    $$
    Where $f_n$ is the father of node $n$ [S1].
3.  **Restricted Adjacency Criterion:** A nephew node $n$ belonging to the tree of root $p_2$ ($T_{p2}$) has a common boundary with block $p_1$ if and only if [S1]:
    $$
    \forall n' \in A_{c_{p2},n}, \text{type}(n') \neq \text{type}(p1)
    $$
    Where $c_{p2}$ is the child of $p_2$ that is an ancestor of $n$ (or $n$ itself if $n$ is a child of $p_2$) [S1]. Essentially, no ancestor of the nephew (up to the sibling of the target) can have the same type as the target block [S1].

**Algorithm Steps:**
1.  Define node types based on the split direction of the father: left/right (x-cut), up/down (y-cut), front/back (z-cut) [S1].
2.  Identify candidate merge partners for each unprocessed leaf node: its brother node and any nephew nodes in the subtrees of the brother's siblings [S1].
3.  Verify adjacency: brothers always share a boundary; for nephews, compute the ancestry set up to the relevant child and verify that no ancestor shares the same type as the target block [S1].
4.  Merge geometry: identify common points on the shared boundary using the Marching Cubes property that a cell edge generates at most one point, making points on common edges identical [S1].
5.  Combine triangle lists and simplify the pasted area using error criterion $E_0$ [S1].
6.  Apply the time lag guard: prevent contraction of an edge $ab$ if the sphere centered on the midpoint of $ab$ with radius $rad(c)$ is not totally included in the merged blocks [S1].
7.  Mark processed: when two brothers are merged, their father node is considered completely processed [S1].
8.  Dump finished components to disk [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Component:
    def __init__(self, triangles, vertices):
        self.triangles = triangles
        self.vertices = vertices

def get_ancestry(node, root):
    """Returns list of nodes from node up to root (inclusive)."""
    ancestry = []
    current = node
    while current is not None:
        ancestry.append(current)
        if current == root:
            break
        current = current.father
    return ancestry

def has_common_boundary(nephew, target):
    """Checks if nephew (in subtree of target's sibling) shares a boundary with target."""
    common_father = nephew.father
    while common_father is not None:
        if common_father.left == target or common_father.right == target or \
           common_father.up == target or common_father.down == target or \
           common_father.front == target or common_father.back == target:
            break
        common_father = common_father.father
    if common_father is None:
        return False
    # Determine the relevant child (sibling of target)
    if common_father.left == target: relevant = common_father.right
    elif common_father.right == target: relevant = common_father.left
    elif common_father.up == target: relevant = common_father.down
    elif common_father.down == target: relevant = common_father.up
    elif common_father.front == target: relevant = common_father.back
    elif common_father.back == target: relevant = common_father.front
    else: return False
    # Verify ancestry condition: no ancestor shares type with target
    for n_prime in get_ancestry(nephew, relevant):
        if n_prime.type == target.type:
            return False
    return True

def merge_and_simplify(comp1, comp2):
    """Merges components, identifies common boundary points, and simplifies."""
    triangles = comp1.triangles + comp2.triangles
    vertices = []
    for tri in triangles:
        for v in tri.vertices:
            if v not in vertices:
                vertices.append(v)
    # Simplify with error criterion E0
    # Time lag guard: prevent edge contraction ab if sphere(midpoint(ab), rad(c)) not in merged blocks
    # Implementation of edge collapse and guard check follows standard geometric predicates [G]
    simplified_triangles = triangles[:] 
    return Component(simplified_triangles, vertices)

def is_father_processed(father_node):
    """Checks if both children of a father node have been processed."""
    if father_node is None: return False
    left_done = father_node.left in processed_set
    right_done = father_node.right in processed_set
    up_done = father_node.up in processed_set
    down_done = father_node.down in processed_set
    front_done = father_node.front in processed_set
    back_done = father_node.back in processed_set
    return (left_done and right_done and up_done and down_done and front_done and back_done)

def restricted_adjacency_merge(leaf_nodes):
    global processed_set
    processed_set = set()
    for node in leaf_nodes:
        if node in processed_set:
            continue
        # Check brother merge
        brother = None
        if node.father:
            if node.father.left == node: brother = node.father.right
            elif node.father.right == node: brother = node.father.left
            elif node.father.up == node: brother = node.father.down
            elif node.father.down == node: brother = node.father.up
            elif node.father.front == node: brother = node.father.back
            elif node.father.back == node: brother = node.father.front
        if brother and brother not in processed_set:
            merged = merge_and_simplify(node.component, brother.component)
            node.component = merged
            processed_set.add(brother)
            if is_father_processed(node.father):
                processed_set.add(node.father)
            continue
        # Check nephew merges
        if node.father:
            for sibling in [node.father.left, node.father.right, node.father.up, node.father.down, node.father.front, node.father.back]:
                if sibling and sibling != node and sibling not in processed_set:
                    # Traverse subtree to find leaves (nephews)
                    stack = [sibling]
                    while stack:
                        curr = stack.pop()
                        if curr.is_leaf:
                            if curr not in processed_set and has_common_boundary(curr, node):
                                merged = merge_and_simplify(node.component, curr.component)
                                node.component = merged
                                processed_set.add(curr)
                                break
                        else:
                            for child in [curr.left, curr.right, curr.up, curr.down, curr.front, curr.back]:
                                if child: stack.append(child)
                    if node.component != comp1: break
    # Dump finished components
    for node in leaf_nodes:
        if node not in processed_set:
            dump_to_disk(node.component)

def dump_to_disk(component):
    """Writes component data to storage."""
    pass
```

**Parameters:**
*   $E_0$: Error criterion for simplification. Typical value: Not given in the sources.
*   $rad(c)$: Radius for the time lag guard. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Numerical Inaccuracy:** Common points on shared boundaries cannot be identified by simple coordinate comparison. Must rely on the Marching Cubes property that a cell edge generates at most one point [S1].
*   **Ancestry Check Failure:** If the ancestry check is not performed correctly, non-adjacent blocks may be merged, leading to topological errors. Specifically, if an ancestor of the nephew has the same type as the target, they do not share a boundary [S1].
*   **Processing Order:** The algorithm requires a specific traversal order (bottom-up) to ensure that when a merge happens, the components are ready. If a brother is merged, the father is considered processed [S1].

## Subtopics

### Boundary Point Identification and Merge Simplification
This variant details the geometric handling of merged surfaces [S1]. It identifies common points extracted in each block lying on their shared boundary [S1]. It merges points belonging to common cell edges, leveraging the property that a cell edge generates only one or zero points, making points on common edges identical [S1]. It simplifies the pasted area using the same error criterion $E_0$ as for surface extraction [S1]. It prevents edge contraction $ab$ during merge simplification as long as the sphere centered on the middle of $ab$ with radius $rad(c)$ is not totally included in the merged blocks [S1]. Finally, it dumps finished components to disk [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Depends on the number of leaves and the depth of the tree for ancestry checks [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Reduces the number of triangles significantly by merging adjacent components [S1]. Avoids waiting for slower blocks by allowing selective merging with nephews [S1]. Improves boundary triangulation quality by applying the time lag method near block boundaries during simplification [S1].
*   **Limitations:** Requires careful implementation of the ancestry check [S1]. Merging is restricted, so not all adjacent components may be merged if they do not meet the strict adjacency criteria [S1].
*   **When to use:** When extracting isosurfaces from large volumes partitioned into blocks, especially when parallel extraction leads to fragmented components that need to be recombined [S1].

## Background
Restricted Adjacency Merging was introduced in the context of parallel extraction and simplification of large isosurfaces using an extended tandem algorithm [S1]. It addresses the issue of non-uniform distribution of iso-surface components in sub-blocks, which can lead to load imbalance and inefficient merging if all brothers are forced to merge [S1].

## Sources
[S1] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
