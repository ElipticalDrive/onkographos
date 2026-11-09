# Error-Based Model Generation (Bottom-Up Fusion)
Also called: DP Removal, Tetrahedra Fusion

## Overview
Error-Based Model Generation is a bottom-up simplification process that approximates a high-resolution tetrahedral volume dataset with fewer tetrahedra by merging child nodes into their parents based on approximation error criteria [S1]. It differs from top-down approaches by starting at the leaf nodes and ascending, ensuring that fusion decisions are driven by local geometric fidelity rather than global hierarchy constraints [S1]. This method builds on the [Multi-Tetra Framework](1183-multi-tetra-framework.md) to selectively retain nodes that meet a precision threshold [S1].

## How it works
**Inputs:**
- A binary tree representation of the volume data, where each node corresponds to a tetrahedron (or a set of tetrahedra) [S1].
- An error threshold $\epsilon$ [S1].

**Outputs:**
- A simplified model consisting of a subset of nodes from the original binary tree [S1].

**Data Structures:**
1. **Binary Tree Array:** An array where each item corresponds to a node in the tree [S1]. Each item stores only the vertex indices of the tetrahedron [S1]. Children addresses are not stored explicitly but are calculated via simple arithmetic [S1].
2. **Dividing Point Structure:** Stores the addresses of all tetrahedra sharing a specific dividing point (vertex) and the maximum approximation error associated with that point [S1].
3. **Dividing Point Table:** A table containing head pointers to chains [S1]. Each chain corresponds to a specific level of the tree [S1]. Each item in a chain is a Dividing Point Structure [S1].

**Algorithm Steps:**

1. **Initialize Error Calculation:**
   Start from the highest resolution level (leaf nodes) and proceed bottom-up [S1]. Do not visit tree nodes directly; instead, visit only the dividing point table [S1].

2. **Process Dividing Points Level-by-Level:**
   For each level of the tree, iterate through all dividing points in the corresponding chain in the Dividing Point Table [S1]:
   
   a. **Compute Current Level Error ($E_{curr}$):**
      For a given dividing point $P$, identify all tetrahedra pairs sharing $P$ [S1]. Calculate the approximation error for each pair [S1]. $E_{curr}$ is the maximum of these errors [S1].
      
   b. **Update Dividing Point Error ($E_{DP}$):**
      The stored error for the dividing point is updated to be the maximum of the current level error and the error already stored from its children:
      $$ E_{DP} = \max(E_{curr}, E_{stored}) $$
      where $E_{stored}$ is the error propagated from lower levels (children) [S1].
      
   c. **Update Parent Error:**
      After processing all dividing points at the current level, update the error of the parent nodes [S1]. The parent's error is derived from the errors of its children (dividing points) [S1].
      
   d. **Ascend:**
      Once all dividing points at the current level are processed, ascend to the next immediate level [S1].

3. **Fusion Criteria Check:**
   Fusion of two child tetrahedra into their parent is allowed only if:
   - The approximation error is below the threshold (implicitly checked during selection) [S1].
   - The topological adjacent relationship is valid (i.e., the children share the dividing point and can be merged without violating topology) [S1].
   - If children cannot be fused or any pair sharing the point cannot be fused, the current fusion fails for that branch [S1].

4. **Model Generation:**
   Select nodes for the final model based on the error threshold $\epsilon$:
   - A node is included in the model if its approximation error $E_{node} < \epsilon$ [S1].
   - A node is included only if its parent's error $E_{parent} > \epsilon$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_based_model_generation(tree_array, dividing_point_table, error_threshold):
    """
    Simplifies a tetrahedral volume model using bottom-up fusion.
    
    Args:
        tree_array: List of Node objects. Each node has `vertex_indices` and `error`.
        dividing_point_table: List of lists. Outer list is levels (bottom to top).
                              Inner list contains DividingPoint objects.
        error_threshold: Precision value for inclusion.
        
    Returns:
        List of node indices included in the simplified model.
    """
    # Initialize all dividing point errors to 0
    for level in dividing_point_table:
        for dp in level:
            dp.max_error = 0.0

    # Process levels from bottom (highest resolution) to top
    for level in reversed(dividing_point_table):
        for dp in level:
            # Step 2a: Compute current level error
            current_level_error = 0.0
            for tetra_pair in dp.tetrahedra_pairs:
                error = compute_approximation_error(tetra_pair)
                if error > current_level_error:
                    current_level_error = error
            
            # Step 2b: Update dividing point error
            dp.max_error = max(current_level_error, dp.max_error)
            
            # Step 2c: Update parent's error
            parent_node = get_parent_node(dp)
            if parent_node is not None:
                parent_node.error = max(parent_node.error, dp.max_error)

    # Step 4: Generate approximate model
    simplified_model = []
    for i, node in enumerate(tree_array):
        parent_node = get_parent_node(node)
        parent_error = parent_node.error if parent_node is not None else float('inf')
        
        # Include node if its error is below threshold and parent's error is above threshold
        if node.error < error_threshold and parent_error > error_threshold:
            simplified_model.append(i)
            
    return simplified_model

def compute_approximation_error(tetrahedron_pair):
    """
    Computes the geometric approximation error for a pair of tetrahedra.
    Evaluates density at points within the tetrahedra and compares the 
    reconstructed surface against the original data [S1].
    """
    # Implementation depends on the specific density evaluation method.
    # Typically calculates maximum deviation between approximated and original isosurface.
    # Not given in the sources.
    return 0.0

def get_parent_node(dividing_point_or_node):
    """
    Returns the parent node of a dividing point or tree node.
    Children addresses are calculated via simple arithmetic as per the framework [S1].
    """
    # Standard heap-like or framework-specific arithmetic calculation [G].
    # Returns parent index or None for root.
    pass
```

**Parameters:**
- **Error Threshold ($\epsilon$):** A precision value used to determine which nodes are included in the final model [S1]. Nodes with error $< \epsilon$ are kept if their parent's error $> \epsilon$ [S1].

**Edge Cases and Pitfalls:**
- **Boundaries:** Dividing points on the boundary of the volume may have fewer adjacent tetrahedra, resulting in a relatively smaller $n$ and affecting the error calculation [S1].
- **Degenerate Input:** If a tetrahedron pair cannot be fused due to topological constraints, the fusion fails for that branch, potentially leading to a less simplified model than expected [S1].

## Subtopics

### [Progressive Hierarchy with Multi-triangulation](6730-progressive-hierarchy-with-multi-triangulation.md)
A dynamic Level-of-Detail (LOD) technique for volume rendering on unstructured grids that creates a progressive hierarchy of tetrahedra stored in a multi-triangulation data structure. It differs from static LOD methods by dynamically updating the mesh to achieve interactive frame rates, building on the bottom-up fusion principles of [Error-Based Model Generation (Bottom-Up Fusion)](1306-error-based-model-generation-bottom-up-fusion.md) by encoding all necessary simplification operations in advance.

## Operating characteristics
- **Time Complexity:** The algorithm visits each dividing point once per level, making it efficient for bottom-up traversal [S1]. The complexity is proportional to the number of dividing points and tetrahedra pairs [S1].
- **Memory Complexity:** Requires storage for the binary tree array, dividing point structures, and the dividing point table [S1]. The binary tree array stores only vertex indices, reducing memory overhead [S1].
- **Performance:** Reduces rendering time geometrically in relation to the degree of reduction, as fewer tetrahedra need to be visited [S1].
- **Strengths:** Local fusion does not have a widespread global effect on complexity, allowing for efficient simplification [S1]. No ambiguity in reconstruction compared to Marching Cubes [S1].
- **Limitations:** The method is static; it does not dynamically update the mesh during rendering unless combined with techniques like Progressive Hierarchy with Multi-triangulation [S1].

## Background
The method was introduced in the paper "Multiresolution tetrahedral framework for visualizing regular volume data" [S1]. It builds on the Multi-Tetra Framework, which replaces each cell with six tetrahedra to avoid the ambiguity of Marching Cubes [S1]. The bottom-up fusion approach is inspired by earlier simplification work [S1].

## Sources
[S1] Multiresolution tetrahedral framework for visualizing regular volume data, https://doi.org/10.5555/266989.267040, fetched 2026-10-30
