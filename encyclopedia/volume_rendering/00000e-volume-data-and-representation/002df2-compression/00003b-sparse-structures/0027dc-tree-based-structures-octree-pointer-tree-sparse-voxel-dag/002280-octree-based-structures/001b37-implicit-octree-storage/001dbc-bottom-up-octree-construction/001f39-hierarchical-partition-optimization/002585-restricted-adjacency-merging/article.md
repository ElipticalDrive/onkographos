# Restricted Adjacency Merging

Also called: Asynchronous merging, Block Merging with Common Border Criterion, Brother/Nephew Merging, Tree-based merging

## Overview
Restricted Adjacency Merging is a load-balancing strategy for parallel isosurface extraction that allows completed sub-blocks to merge with available neighbors beyond immediate siblings, provided they share a geometric common boundary [S1, S2]. It solves the problem of idle processors waiting for slower sibling blocks to finish extraction, which creates bottlenecks when isosurface components are unevenly distributed across the spatial decomposition [S1, S2]. It sets itself apart from strict hierarchical merging by introducing a "common border" criterion based on node ancestry types, enabling asynchronous merging of a block with its brother or a brother's child (nephew) [S1, S2].

## How it works
**Inputs:** A set of extracted surface meshes from independent sub-blocks of a volume dataset; the hierarchical tree structure of the spatial decomposition (octree or similar).
**Outputs:** A unified, manifold isosurface mesh dumped to disk.

**Data Structures:**
*   **Node Type:** An enumeration representing the position of a node relative to its father's split plane.
    *   X-split: `left`, `right`
    *   Y-split: `up`, `down`
    *   Z-split: `front`, `bottom` (or `back`)
*   **Ancestry Set:** For a node $n$ in a tree rooted at $p$, the set of nodes containing $n$ and its recursive fathers up to $p$.

**Steps:**

1.  **Define Node Types:**
    Determine the type of each node based on the split direction of its father. If a node $p$ is split along the x-axis, its children are typed `left` and `right`. Similarly, y-split yields `up`/`down`, and z-split yields `front`/`bottom` [S1, S2].

2.  **Define Opposite Operator:**
    Establish the inverse relationship between types to identify opposing sides of a split plane:
    $$
    \begin{aligned}
    \text{left} &= \text{right}, & \text{right} &= \text{left} \\
    \text{up} &= \text{down}, & \text{down} &= \text{up} \\
    \text{front} &= \text{bottom}, & \text{bottom} &= \text{front}
    \end{aligned}
    $$
    [S1, S2]

3.  **Identify Common Points (Boundary Identification):**
    Identify vertices shared between adjacent blocks. Due to numerical inaccuracies in vertex extraction, coordinate comparison is unreliable. Instead, use the property of the Marching Cubes algorithm: a cell edge can generate at most one point. Therefore, points lying on common cell edges between two blocks are geometrically identical and can be merged directly [S1, S2].

4.  **Check Restricted Adjacency (Merge Eligibility):**
    Determine if a completed block $p_1$ can merge with another block $p_2$ (brother) or $n$ (nephew, i.e., a child of $p_1$'s brother).
    
    *   **Brother Merge:** Two brothers $p_1$ and $p_2$ can always merge if both are finished [S2].
    *   **Nephew/Brother-Child Merge:** A node $p$ belonging to tree $T_{p1}$ has a common border with $p_2$ if:
        $$ \forall p' \in A_{p, p1}, \text{type}(p') \neq \text{type}(p2) $$
        where $A_{p, p1}$ is the ancestry of $p$ up to $p1$ [S1].
        
        A nephew node $n$ belonging to $T_{c_{p2}}$ (the tree of the brother of $p_1$) has a common boundary with $p_1$ if:
        $$ \forall n' \in A_{c_{p2}, n}, \text{type}(n') \neq \text{type}(p1) $$
        [S2]
        
        This criterion ensures that the blocks share a true geometric boundary and do not overlap or touch only at corners/edges in a way that violates manifold constraints. Implementation involves iterative tree traversal checking the type condition against successive fathers [S2].

5.  **Simplify Pasted Area:**
    After merging the meshes, simplify the "pasted area" (the interface) using the same error criteria $E_0$ used during extraction. Prevent edge contractions if the "time lag sphere" (centered on the edge midpoint with radius $rad(c)$) is not fully included in the merged blocks. This preserves surface quality and prevents artifacts [S1, S2].

6.  **Mark Processed:**
    When two brothers are merged, their father node is considered completely processed. This status propagates up the tree, allowing higher-level merges [S2].

7.  **Dump:**
    Dump the finished, merged components to disk [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_node_type(node):
    """Returns the type of a node relative to its father's split."""
    if node.father is None:
        return None
    split_dir = node.father.split_direction
    if split_dir == 'x':
        return 'left' if node.position == node.father.left_child else 'right'
    elif split_dir == 'y':
        return 'up' if node.position == node.father.up_child else 'down'
    elif split_dir == 'z':
        return 'front' if node.position == node.father.front_child else 'bottom'

def get_ancestry(node, root):
    """Returns the set of nodes from node up to root (inclusive)."""
    ancestry = set()
    current = node
    while current is not None and current != root:
        ancestry.add(current)
        current = current.father
    ancestry.add(root) # Include root as per definition A_{n,p} containing p
    return ancestry

def has_common_border_brother(node1, node2):
    """Check if two brother nodes have a common border (always true for brothers in this context)."""
    # Brothers share the same father and are on opposite sides of the split.
    return node1.father == node2.father

def has_common_border_nephew(nephew_node, uncle_node):
    """
    Check if a nephew node (child of uncle's brother) has a common border with the uncle.
    uncle_node is p1. nephew_node is n in T_{c_p2} where p2 is brother of p1.
    """
    # Find the brother of the uncle (the father of the nephew)
    # Actually, the criterion is defined relative to the target block p1.
    # Let p1 = uncle_node. Let n = nephew_node.
    # n belongs to the tree of p2, where p2 is the brother of p1.
    # We need to verify ancestry of n up to p2.
    
    # Identify p2: The father of nephew_node is the brother of uncle_node?
    # No, nephew_node is a child of p2. p2 is the brother of p1.
    # So, p2 = nephew_node.father.
    # We must ensure p2 is indeed the brother of p1.
    
    p2 = nephew_node.father
    if p2.father != uncle_node.father:
        return False # Not siblings' children
    
    # Check ancestry of n up to p2
    ancestry_n = get_ancestry(nephew_node, p2)
    
    type_p1 = get_node_type(uncle_node)
    
    for n_prime in ancestry_n:
        if get_node_type(n_prime) == type_p1:
            return False # Type matches, no common border (overlap or corner touch)
            
    return True

def merge_meshes(mesh1, mesh2):
    """
    Merge two meshes sharing a boundary.
    Uses Marching Cubes property: points on common cell edges are identical.
    """
    # 1. Identify common points via edge topology, not coordinate comparison.
    # 2. Combine vertices and faces.
    # 3. Simplify pasted area using error criteria E0.
    #    Prevent edge contraction if time lag sphere is not fully included.
    pass

def restricted_adjacency_merge(completed_blocks):
    """
    Main merge loop.
    completed_blocks: List of (node, mesh) tuples where extraction is done.
    """
    while completed_blocks:
        # Pick a completed block p1
        p1, mesh1 = completed_blocks.pop(0)
        
        merged = False
        
        # Try to merge with brothers first (simpler, always valid if brother done)
        # In a parallel scheduler, we look for available neighbors.
        
        # Check for brother p2
        brother = p1.brother
        if brother and brother.is_completed:
            # Brothers always have common border
            mesh2 = brother.mesh
            merged_mesh = merge_meshes(mesh1, mesh2)
            # Mark father as processed
            p1.father.mark_processed()
            dump_to_disk(merged_mesh)
            merged = True
            
        if not merged:
            # Try to merge with nephews (children of brothers)
            # Iterate through available completed blocks to find potential nephews
            for i, (p2, mesh2) in enumerate(completed_blocks):
                # Check if p2 is a nephew of p1
                # p2 is nephew if p2.father is brother of p1
                if p2.father == p1.brother:
                    if has_common_border_nephew(p2, p1):
                        merged_mesh = merge_meshes(mesh1, mesh2)
                        # Simplify pasted area
                        simplify_pasted_area(merged_mesh)
                        dump_to_disk(merged_mesh)
                        completed_blocks.pop(i)
                        merged = True
                        break
        
        if not merged:
            # If no merge possible, keep in queue or dump if isolated?
            # Sources imply dumping finished components. 
            # If it can't merge, it might be an isolated component or waiting.
            # The text says "dump finished components on the disk" as the last step.
            # Usually, unmergeable components are dumped as is.
            dump_to_disk(mesh1)
```

**Edge Cases and Pitfalls:**
*   **Numerical Inaccuracy:** Do not use coordinate comparison to identify common points on boundaries; use the Marching Cubes edge property [S1, S2].
*   **Ancestry Check:** When merging a nephew, it is critical to check *all* ancestors of the nephew up to the brother node. If any ancestor shares the same type as the target block, they do not share a common boundary (e.g., they touch only at a corner or are on the same side of a split) [S2].
*   **Processing Order:** When two brothers are merged, their father is marked completely processed. This status must be propagated to allow higher-level merges (e.g., merging the father with its brother) [S2].

## Subtopics

### [Boundary Point Identification and Merge Simplification](5296-boundary-point-identification-and-merge-simplification.md)
Boundary Point Identification and Merge Simplification is a post-processing step in parallel isosurface extraction that resolves geometric inconsistencies at the interfaces between independently processed sub-blocks. It solves the problem of non-manifold geometry and visual artifacts caused by numerical discrepancies in vertex extraction when adjacent blocks generate slightly different coordinates for the same boundary points. It sets itself apart from general mesh simplification by specifically targeting the "pasted area" where blocks meet, using topological properties of the marching cubes algorithm to identify coincident points and applying a conservative error criterion to ensure the merged surface remains within the original extraction tolerance.

## Operating characteristics
*   **Time Complexity:** The merge eligibility check requires traversing the ancestry of nodes, which is $O(h)$ where $h$ is the height of the tree. The actual mesh merging and simplification depend on the number of triangles at the boundary.
*   **Memory Complexity:** Requires storing the tree structure and the meshes of completed blocks until they are merged or dumped.
*   **Performance:** Improves load balancing in parallel environments by reducing idle time caused by uneven distribution of isosurface complexity across sub-blocks [S1, S2].
*   **Strengths:** Asynchronous execution; handles non-uniform data distribution better than strict brother-merging.
*   **Limitations:** Requires careful implementation of the ancestry type check to avoid invalid merges. The "time lag" simplification adds computational overhead to the merge step [S1].

## Background
Restricted Adjacency Merging was introduced in the context of parallel isosurface extraction for very large geophysical datasets. It was developed to address load imbalance issues in distributed memory architectures (e.g., clusters managed by LSF-HPC) where the extraction time of sub-blocks varied significantly [S1, S2]. The method builds on the extended tandem algorithm and Marching Cubes isosurface extraction [S1, S2].

## Sources
[S1] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S2] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
