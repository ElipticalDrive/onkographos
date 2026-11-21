# Error-Based Model Generation (Bottom-Up Fusion)
Also called: DP Removal, Tetrahedra Fusion

## Overview
Error-Based Model Generation is a bottom-up simplification process that approximates a high-resolution tetrahedral volume dataset with fewer tetrahedra by merging child nodes into their parents based on approximation error criteria [S1]. It solves the problem of rendering performance degradation in high-resolution multiresolution volumes by geometrically reducing the number of elements visited during rendering [S1]. It differs from static LOD methods by dynamically evaluating fusion criteria at every dividing point in a bottom-up traversal, ensuring that topological constraints and geometric errors are respected before any merge occurs [S1].

## How it works
**Inputs:**
- A Multi-Tetra Framework binary tree representing the volume data [S1].
- An error threshold $\epsilon$ [S1].

**Outputs:**
- A simplified tetrahedral model consisting of nodes whose approximation error is less than $\epsilon$ while their parent's error exceeds $\epsilon$ [S1].

**Data Structures:**
1.  **Binary Tree Array:** An array where each item corresponds to a tree node. Each item stores only the vertex indices of the tetrahedron. Children addresses are not stored explicitly but are calculated via simple arithmetic (e.g., standard heap indexing or derived from the array position) [S1].
2.  **Dividing Point Structure:** Stores the addresses of all tetrahedra sharing a specific dividing point and the maximum approximation error associated with that point [S1].
3.  **Dividing Point Table:** A table containing head pointers to chains. Each chain corresponds to a specific level of the tree. Each item in a chain is a Dividing Point Structure [S1].

**Algorithm Steps:**

1.  **Initialize Error Calculation:**
    Start from the highest resolution level (leaf nodes) and proceed bottom-up. Do not visit tree nodes directly; instead, visit only the dividing point table [S1].

2.  **Compute Current Level Error:**
    For each dividing point $dp$ at the current level:
    - Identify all tetrahedra pairs sharing $dp$.
    - Calculate the approximation error for each pair. The error is based on geometric approximation (e.g., density evaluation error within the tetrahedron) [S1].
    - Let $E_{current}(dp)$ be the maximum error among all tetrahedra pairs sharing $dp$ [S1].

3.  **Update Dividing Point Error:**
    Let $E_{stored}(dp)$ be the error already stored in the Dividing Point Structure from processing the children levels.
    Update the dividing point's error:
    $$ E(dp) = \max(E_{current}(dp), E_{stored}(dp)) $$
    [S1]

4.  **Propagate to Parent:**
    After processing all dividing points at the current level, update the error of their parent nodes. The parent's error is derived from the errors of the dividing points associated with its children [S1].

5.  **Check Fusion Criteria:**
    For a fusion to succeed at a dividing point:
    - The children must be fusable (i.e., they share the same parent).
    - All tetrahedra pairs sharing the dividing point must be fusable.
    - If any pair cannot be fused or if the error exceeds the local tolerance implied by the structure, the fusion fails for that point [S1].
    - If fusion fails, the current level's error remains, and the parent's error is updated accordingly to reflect the inability to simplify further at that location [S1].

6.  **Ascend Levels:**
    After all dividing points at the current level are processed, ascend to the next immediate level and repeat steps 2–5 [S1].

7.  **Generate Approximate Model:**
    After the bottom-up traversal is complete, select nodes for the final model:
    - Include a node if its approximation error $E(node) < \epsilon$.
    - Exclude the node if its parent's error $E(parent) > \epsilon$.
    - This ensures a smooth transition between levels of detail [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_based_model_generation(tree_array, dividing_point_table, epsilon):
    """
    tree_array: Array of nodes, each containing vertex indices.
    dividing_point_table: Dict mapping level -> list of DividingPoint objects.
    epsilon: Error threshold.
    """
    
    # Step 1: Bottom-up traversal
    levels = sorted(dividing_point_table.keys(), reverse=True)
    
    for level in levels:
        dividing_points = dividing_point_table[level]
        
        for dp in dividing_points:
            # Step 2: Compute current level error
            # Identify tetrahedra pairs sharing this dividing point
            tetra_pairs = get_tetrahedra_pairs_sharing_point(dp)
            
            max_pair_error = 0.0
            for pair in tetra_pairs:
                error = calculate_approximation_error(pair)
                if error > max_pair_error:
                    max_pair_error = error
            
            # Step 3: Update dividing point error
            # dp.error stores the max error from children levels
            dp.error = max(dp.error, max_pair_error)
            
            # Step 4: Propagate to parent
            # The parent's error is influenced by the dividing points of its children
            # In the Multi-Tetra framework, parent error is updated based on 
            # the errors of the dividing points associated with the child nodes.
            parent_node = get_parent_node(dp)
            if parent_node:
                update_parent_error(parent_node, dp)
    
    # Step 7: Generate model
    simplified_nodes = []
    for node in tree_array:
        if node.error < epsilon:
            # Check if parent exceeds threshold to ensure LOD transition
            parent = get_parent_node(node)
            if parent is None or parent.error > epsilon:
                simplified_nodes.append(node)
                
    return simplified_nodes

def calculate_approximation_error(tetrahedron_pair):
    """
    Calculates the geometric approximation error for a pair of tetrahedra.
    This involves evaluating the density function within the tetrahedra
    and comparing it to the parent representation.
    """
    # Implementation depends on the specific density evaluation method
    # referenced in [S1], typically involving interpolation or sampling.
    # [G] Placeholder for specific density evaluation formula.
    return 0.0

def update_parent_error(parent_node, child_dp):
    """
    Updates the parent node's error based on the child dividing point's error.
    The parent error is the maximum of its children's errors.
    """
    if parent_node.error < child_dp.error:
        parent_node.error = child_dp.error
```

**Parameters:**
- **Error Threshold ($\epsilon$):** A precision value used to determine which nodes are included in the final model. Nodes with error $< \epsilon$ are kept if their parent's error $> \epsilon$ [S1].

**Edge Cases and Pitfalls:**
- **Fusion Failure:** If any pair of tetrahedra sharing a dividing point cannot be fused, the current fusion fails, and the error is not reduced at that point [S1].
- **Boundary Conditions:** Dividing points on the boundaries of the volume may have fewer adjacent tetrahedra, resulting in a relatively smaller $n$ (number of pairs), which affects the local error calculation [S1].
- **Precision:** The error calculation must be robust to avoid numerical instability when comparing small differences in density values [S1].

## Subtopics

### [Progressive Hierarchy with Multi-triangulation](6730-progressive-hierarchy-with-multi-triangulation.md)
Progressive Hierarchy with Multi-triangulation is a dynamic Level-of-Detail (LOD) technique for volume rendering on unstructured grids that creates a progressive hierarchy of tetrahedra stored in a multi-triangulation data structure. It solves the problem of interactive rendering performance by allowing the mesh to be dynamically updated and selectively refined to a target LOD using selection heuristics. It differs from static LOD methods by encoding all edge-collapse operations required to reduce the mesh to a base state, enabling on-the-fly reconstruction.

## Operating characteristics
- **Time Complexity:** The algorithm visits each dividing point once per level, making it linear with respect to the number of dividing points in the multiresolution structure [S1].
- **Memory Complexity:** Requires storage for the binary tree array, dividing point structures, and the dividing point table. The binary tree array stores only vertex indices, minimizing memory overhead [S1].
- **Performance:** Rendering time is reduced geometrically in relationship to the degree of reduction, as fewer tetrahedra are visited [S1].
- **Strengths:** Bottom-up approach ensures that fusion criteria are met locally before global simplification. The use of dividing points allows for efficient error propagation without traversing the entire tree [S1].
- **Limitations:** The method is specific to the Multi-Tetra Framework and may not be directly applicable to other volume representations. The error calculation depends on the density evaluation method, which may vary [S1].

## Background
The method was introduced in the context of the Multi-Tetra Framework for visualizing regular volume data [S1]. It builds on the idea of replacing regular grid cells with six tetrahedra and using a binary tree to represent multiresolution levels [S1]. The approach is inspired by bottom-up simplification techniques such as those described in [LH91] [S1]. It is used for isosurface extraction via the Marching Tetrahedra method [ZCT95], which avoids the ambiguities present in Marching Cubes [S1].

## Sources
[S1] Multiresolution tetrahedral framework for visualizing regular volume data, https://doi.org/10.5555/266989.267040, fetched 2026-10-30
