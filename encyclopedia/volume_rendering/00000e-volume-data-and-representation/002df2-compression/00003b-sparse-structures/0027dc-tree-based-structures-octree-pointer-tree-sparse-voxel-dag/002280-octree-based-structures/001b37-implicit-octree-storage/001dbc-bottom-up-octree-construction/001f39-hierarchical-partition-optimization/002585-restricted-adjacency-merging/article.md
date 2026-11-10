# Restricted Adjacency Merging

Also called: Asynchronous merging, Block Merging with Common Border Criterion, Brother/Nephew Merging, Tree-based merging

## Overview
Restricted Adjacency Merging is a load-balancing strategy for parallel isosurface extraction that allows completed sub-blocks to merge with available neighbors beyond immediate siblings, provided they share a geometric common boundary [S1, S2]. It solves the problem of idle processors waiting for slower sibling blocks to finish extraction, which creates bottlenecks when isosurface components are unevenly distributed across the spatial decomposition [S1, S2]. It sets itself apart from strict hierarchical merging by introducing a "common border" criterion based on node ancestry types, enabling asynchronous merging of a block with its brother or a brother's child (nephew) [S1, S2].

## How it works
The method takes as input the extracted surface meshes from independent sub-blocks of a volume dataset and the hierarchical tree structure of the spatial decomposition [S1, S2]. It outputs a unified, manifold isosurface mesh that is dumped to disk [S1].

**Data Structures:**
*   **Node Type:** An enumeration representing the position of a node relative to its father's split plane [S1, S2].
    *   X-split: `left`, `right`
    *   Y-split: `up`, `down`
    *   Z-split: `front`, `bottom` (Source [S1]) or `front`, `back` (Source [S2]) [S1, S2].
*   **Ancestry Set:** For a node $n$ in a tree rooted at $p$, the set of nodes containing $n$ and its recursive fathers up to $p$ [S2].

**Steps:**

1.  **Define Node Types:** Determine the type of each node based on the split direction of its father. If a node is split along the x-axis, its children are typed `left` and `right`. Similarly, y-split yields `up`/`down`, and z-split yields `front`/`bottom` (or `front`/`back`) [S1, S2].
2.  **Define Opposite Operator:** Establish the inverse relationship between types to identify opposing sides of a split plane:
    $$ \text{left} = \text{right}, \quad \text{right} = \text{left} $$
    $$ \text{up} = \text{down}, \quad \text{down} = \text{up} $$
    $$ \text{front} = \text{bottom}, \quad \text{bottom} = \text{front} $$
    (Note: [S2] uses `back` instead of `bottom` for the z-axis) [S1, S2].
3.  **Identify Common Points:** Identify vertices shared between adjacent blocks. Due to numerical inaccuracies in vertex extraction, coordinate comparison is unreliable. Instead, use the property of the Marching Cubes algorithm: a cell edge can generate at most one point. Therefore, points lying on common cell edges between two blocks are geometrically identical and can be merged directly [S1].
4.  **Check Restricted Adjacency:** Determine if a completed block $p_1$ can merge with another block $p_2$ (brother) or $n$ (nephew, i.e., a child of $p_1$'s brother).
    *   **Brother Merge:** Two brothers $p_1$ and $p_2$ can merge if both are finished [S2].
    *   **Nephew/Brother-Child Merge:** A node $p$ belonging to tree $T_{p1}$ has a common border with $p_2$ if:
        $$ \forall p' \in A_{p, p1}, \text{type}(p') \neq \text{type}(p2) $$
        where $A_{p, p1}$ is the ancestry of $p$ up to $p1$ [S1].
        A nephew node $n$ belonging to $T_{c_{p2}}$ (the tree of the brother of $p_1$) has a common boundary with $p_1$ if:
        $$ \forall n' \in A_{c_{p2}, n}, \text{type}(n') \neq \text{type}(p1) $$
        [S2]. This criterion ensures that the blocks share a true geometric boundary. Implementation involves iterative tree traversal checking the type condition against successive fathers [S2].
5.  **Simplify Pasted Area:** After merging the meshes, simplify the "pasted area" (the interface) using the same error criteria $E_0$ used during extraction. Prevent edge contractions if the "time lag sphere" (centered on the edge midpoint with radius $rad(c)$) is not fully included in the merged blocks [S1].
6.  **Mark Processed:** When two brothers are merged, their father node is considered completely processed [S2].
7.  **Dump:** Dump the finished, merged components to disk [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_node_type(node):
    """Returns the type of a node relative to its father's split direction."""
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
    """Returns the set of nodes from node up to root (inclusive), matching A_{root, node}."""
    ancestry = set()
    current = node
    while current is not None and current != root:
        ancestry.add(current)
        current = current.father
    ancestry.add(root)
    return ancestry

def has_common_border_brother(node1, node2):
    """Brothers share the same father and are on opposite sides of the split."""
    return node1.father is not None and node1.father == node2.father

def has_common_border_nephew(nephew_node, target_node):
    """
    Check if a nephew node has a common border with target_node (p1).
    nephew_node is n in T_{c_p2} where p2 is the brother of target_node.
    Criterion: type(n') != type(target_node) for all n' in ancestry of n up to p2.
    """
    p2 = nephew_node.father
    if p2.father != target_node.father:
        return False
    ancestry_n = get_ancestry(nephew_node, p2)
    type_target = get_node_type(target_node)
    for n_prime in ancestry_n:
        if get_node_type(n_prime) == type_target:
            return False
    return True

def identify_common_points(mesh1, mesh2):
    """Identify shared vertices using Marching Cubes edge property."""
    common_vertices = set()
    for v in mesh1.vertices:
        if v.edge_index in mesh2.edge_to_vertex:
            common_vertices.add(v)
    return common_vertices

def apply_time_lag_simplification(mesh, merged_blocks):
    """Simplify pasted area using error criteria E0 and time lag sphere check."""
    simplified = mesh
    for edge in simplified.edges:
        midpoint = (edge.v1.position + edge.v2.position) / 2.0
        sphere = Sphere(center=midpoint, radius=rad(edge))
        if not sphere.is_fully_contained_in(merged_blocks):
            continue
        # Apply edge contraction logic...
    return simplified

def merge_meshes(mesh1, mesh2, common_pts):
    """Combine two meshes, deduplicating vertices at the common boundary."""
    merged_vertices = list(mesh1.vertices) + list(mesh2.vertices)
    for v1, v2 in common_pts:
        merged_vertices.remove(v2)
    merged_faces = list(mesh1.faces) + list(mesh2.faces)
    return Mesh(vertices=merged_vertices, faces=merged_faces)

def dump_to_disk(mesh, filepath):
    """Write the final mesh geometry to a file."""
    with open(filepath, 'wb') as f:
        f.write(mesh.serialize())

def restricted_adjacency_merge(completed_blocks):
    """
    Main merge loop.
    completed_blocks: List of (node, mesh) tuples where extraction is done.
    """
    while completed_blocks:
        p1, mesh1 = completed_blocks.pop(0)
        merged = False

        # Try to merge with brother
        if p1.brother and p1.brother.is_completed:
            p2, mesh2 = p1.brother.data
            common_pts = identify_common_points(mesh1, mesh2)
            merged_mesh = merge_meshes(mesh1, mesh2, common_pts)
            merged_mesh = apply_time_lag_simplification(merged_mesh, [p1, p2])
            dump_to_disk(merged_mesh, "output.obj")
            p1.father.mark_processed()
            merged = True

        if not merged:
            # Try to merge with nephews
            for i, (p2, mesh2) in enumerate(completed_blocks):
                if has_common_border_nephew(p2, p1):
                    common_pts = identify_common_points(mesh1, mesh2)
                    merged_mesh = merge_meshes(mesh1, mesh2, common_pts)
                    merged_mesh = apply_time_lag_simplification(merged_mesh, [p1, p2])
                    dump_to_disk(merged_mesh, "output.obj")
                    completed_blocks.pop(i)
                    merged = True
                    break

        if not merged:
            dump_to_disk(mesh1, "output.obj")
```

**Edge Cases and Pitfalls:**
*   **Numerical Inaccuracy:** Do not use coordinate comparison to identify common points on boundaries; use the Marching Cubes edge property [S1].
*   **Ancestry Check:** When merging a nephew, it is critical to check *all* ancestors of the nephew up to the brother node. If any ancestor shares the same type as the target block, they do not share a common boundary (e.g., they touch only at a corner or are on the same side of a split) [S2].
*   **Processing Order:** When two brothers are merged, their father is marked completely processed [S2].
*   **Z-split Terminology:** Sources disagree on the z-axis split labels: [S1] uses `front`/`bottom`, while [S2] uses `front`/`back` [S1, S2].

## Subtopics

### [Boundary Point Identification and Merge Simplification](5296-boundary-point-identification-and-merge-simplification.md)
Boundary Point Identification and Merge Simplification is a post-processing step in parallel isosurface extraction that resolves geometric inconsistencies at the interfaces between independently processed sub-blocks. It solves the problem of visual artifacts caused by numerical discrepancies in vertex extraction when adjacent blocks generate slightly different coordinates for the same boundary points. It sets itself apart from general mesh simplification by specifically targeting the "pasted area" where blocks meet, using topological properties of the marching cubes algorithm to identify coincident points and applying a conservative error criterion to ensure the merged surface remains within the original extraction tolerance.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Improves load balancing in parallel environments by reducing idle time caused by uneven distribution of isosurface complexity across sub-blocks [S1, S2].
*   **Strengths:** Asynchronous execution; handles non-uniform data distribution better than strict brother-merging [S1, S2].
*   **Limitations:** Requires careful implementation of the ancestry type check to avoid invalid merges [S2].

## Background
Restricted Adjacency Merging was introduced in the context of parallel isosurface extraction for very large geophysical datasets. It was developed to address load imbalance issues in distributed memory architectures (e.g., clusters managed by LSF-HPC) where the extraction time of sub-blocks varied significantly [S1, S2]. The method builds on the extended tandem algorithm and Marching Cubes isosurface extraction [S1, S2].

## Sources
[S1] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S2] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
