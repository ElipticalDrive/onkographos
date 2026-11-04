# Error-Based Model Generation (Bottom-Up Fusion)

Also called: DP Removal, Tetrahedra Fusion

## Overview
This method performs a bottom-up simplification of the Multi-Tetra Framework to approximate high-resolution volume data with fewer tetrahedra, thereby reducing rendering time [S1]. It merges child tetrahedra into their parent node if the geometric approximation error remains within a specified threshold, solving the trade-off between detail and performance. Unlike top-down approaches, it starts from the leaf nodes and ascends, ensuring that fusion decisions are based on accumulated local errors [S1].

## How it works

### Inputs and Outputs
*   **Input**: A Multi-Tetra binary tree representing the volume data, an error threshold $\epsilon$, and the vertex values associated with the tree nodes.
*   **Output**: A simplified set of tetrahedra (nodes) that approximate the original data within the error bound $\epsilon$.

### Data Structures
The implementation relies on three specific structures defined in the source [S1]:

1.  **Binary Tree Array**: A flat array where each index corresponds to a tree node. Each entry stores only the **vertex indices** of the tetrahedron. Children addresses are not stored explicitly but are calculated via arithmetic (e.g., if node $i$ is at index $k$, children are at $2k+1$ and $2k+2$, or similar standard heap indexing, though the exact arithmetic is not specified beyond "simple calculation" [S1]).
2.  **Dividing Point Structure**: A record associated with a specific geometric vertex (dividing point) in the volume grid. It contains:
    *   A list of addresses (indices) of all tetrahedra sharing this dividing point.
    *   A field for the **maximum approximation error** computed for this point.
3.  **Dividing Point Table**: A table containing head pointers to chains. Each chain corresponds to a specific level of the tree hierarchy. Each item in a chain is a Dividing Point Structure. This allows efficient traversal by level without visiting tree nodes directly [S1].

### Algorithm Steps

1.  **Initialization**:
    *   Construct the Dividing Point Table. Populate the chains with Dividing Point Structures for all unique dividing points in the volume.
    *   For each structure, identify all tetrahedra sharing that point and store their addresses [S1].
    *   Initialize the error field in each Dividing Point Structure to 0 or infinity, depending on implementation, but the source implies errors are accumulated [S1].

2.  **Bottom-Up Error Calculation**:
    *   Iterate through the tree levels from the highest (leaves) to the root.
    *   For each level, iterate through all Dividing Point Structures in the corresponding chain in the Dividing Point Table [S1].
    *   For each dividing point $P$:
        *   Let $T_P$ be the set of tetrahedra sharing point $P$.
        *   Calculate the **current level error** $E_{curr}(P)$ as the maximum approximation error among all tetrahedra pairs in $T_P$ that share $P$. The approximation error for a tetrahedron is derived from the difference between the actual data values and the interpolated values within the tetrahedron (typically using linear interpolation based on vertex values) [S1].
        *   Let $E_{stored}(P)$ be the error already stored in the structure from processing its children.
        *   Update the stored error: $E_{new}(P) = \max(E_{curr}(P), E_{stored}(P))$ [S1].
        *   If any pair of tetrahedra sharing $P$ cannot be fused (i.e., their combined error exceeds limits or topological constraints fail), mark the fusion for $P$ as failed [S1].
    *   After processing all dividing points at the current level, update the error fields of the parent dividing points based on the results from the current level. This ensures that if children cannot be fused, the parent's error reflects this failure [S1].
    *   Ascend to the next immediate level and repeat until the root is processed [S1].

3.  **Model Generation**:
    *   Traverse the tree (or the Dividing Point Table structures) to select the final set of tetrahedra.
    *   A node (tetrahedron) is included in the final model if:
        *   Its approximation error is less than the error threshold $\epsilon$.
        *   Its parent's approximation error exceeds the threshold $\epsilon$ [S1].
    *   This criterion ensures that the finest possible resolution is used where the data changes rapidly (high error), while coarser resolution is used where the data is smooth (low error) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_based_model_generation(tree_array, vertex_values, error_threshold):
    """
    Simplifies the Multi-Tetra framework using bottom-up fusion.
    
    Args:
        tree_array: Array of nodes, each storing vertex indices.
        vertex_values: Array of scalar values for each vertex.
        error_threshold: Maximum allowed approximation error.
        
    Returns:
        List of indices of tetrahedra to keep.
    """
    
    # 1. Build Dividing Point Table
    # Map each unique vertex index to a list of tetrahedra indices sharing it
    dividing_points = {} 
    for tet_idx, vertices in enumerate(tree_array):
        for v in vertices:
            if v not in dividing_points:
                dividing_points[v] = []
            dividing_points[v].append(tet_idx)
            
    # Create structures with error fields
    dp_structures = {v: {'tetrahedra': tets, 'error': 0.0} for v, tets in dividing_points.items()}
    
    # Organize by level (assuming we can determine level from tree structure or input)
    # For simplicity, assume levels are processed from max_depth down to 0
    levels = get_levels_from_tree(tree_array)
    
    # 2. Bottom-Up Error Calculation
    for level in reversed(levels):
        # Get all dividing points at this level
        dp_at_level = get_dividing_points_at_level(level)
        
        for dp in dp_at_level:
            struct = dp_structures[dp]
            tets = struct['tetrahedra']
            
            # Calculate current level error: max error among tetrahedra sharing this point
            current_level_error = 0.0
            for tet_idx in tets:
                # Calculate approximation error for this tetrahedron
                # Error = max(|actual_value - interpolated_value|) over the tetrahedron volume
                # Interpolation is linear based on vertex_values
                tet_error = calculate_tetrahedron_error(tet_idx, tree_array, vertex_values)
                if tet_error > current_level_error:
                    current_level_error = tet_error
            
            # Update stored error: max of current level and children's accumulated error
            struct['error'] = max(current_level_error, struct['error'])
            
            # Note: The source mentions updating parent's error after processing.
            # In a bottom-up pass, the parent's error field will be updated when 
            # the parent's level is processed, or via a separate propagation step.
            # Here, we assume the 'error' field in the structure represents the 
            # maximum error associated with the geometric point, which influences 
            # the parent's decision.
            
    # 3. Model Generation
    # Select nodes where error < threshold AND parent error > threshold
    kept_tetrahedra = []
    
    for tet_idx, vertices in enumerate(tree_array):
        # Determine the error associated with this tetrahedron
        # The source says "nodes with their approximation errors less than the error threshold"
        # We use the max error of the dividing points within this tetrahedron as the node's error
        tet_error = 0.0
        for v in vertices:
            if v in dp_structures:
                if dp_structures[v]['error'] > tet_error:
                    tet_error = dp_structures[v]['error']
        
        # Find parent index
        parent_idx = get_parent_index(tet_idx)
        
        if parent_idx is not None:
            # Get parent's error
            parent_vertices = tree_array[parent_idx]
            parent_error = 0.0
            for v in parent_vertices:
                if v in dp_structures:
                    if dp_structures[v]['error'] > parent_error:
                        parent_error = dp_structures[v]['error']
            
            # Apply selection criteria
            if tet_error < error_threshold and parent_error > error_threshold:
                kept_tetrahedra.append(tet_idx)
        else:
            # Root node
            if tet_error < error_threshold:
                kept_tetrahedra.append(tet_idx)
                
    return kept_tetrahedra

def calculate_tetrahedron_error(tet_idx, tree_array, vertex_values):
    """
    Calculates the maximum approximation error for a tetrahedron.
    Assumes linear interpolation.
    """
    vertices = tree_array[tet_idx]
    # Get values at vertices
    vals = [vertex_values[v] for v in vertices]
    
    # Sample points inside the tetrahedron (e.g., centroid, or dense grid)
    # For exact implementation, one might check the error at the centroid
    # or use a bound based on the gradient.
    # The source does not specify the exact sampling method, only "approximation error".
    # A common approach is the difference between the actual value at the centroid
    # and the interpolated value at the centroid.
    
    centroid = average(vertices) # Geometric centroid
    actual_val = interpolate_at_point(centroid, vertices, vals) # This requires geometric coords
    # Note: The source does not provide geometric coordinates, only vertex indices.
    # In a real implementation, geometric coordinates are needed.
    # [G] Assuming a function get_coords(vertex_index) exists.
    
    # Simplified error metric: max difference between vertex values?
    # No, approximation error usually refers to interpolation error.
    # Without geometric data, we cannot compute this exactly.
    # [G] Placeholder for geometric interpolation error calculation.
    return 0.0 # Placeholder
```

### Parameters
*   **Error Threshold ($\epsilon$)**: A precision value determining the fidelity of the simplified model. Nodes with error $< \epsilon$ are kept if their parent's error $> \epsilon$ [S1].

### Edge Cases and Pitfalls
*   **Precision**: The approximation error calculation must be robust. If the error metric is not accurate, the simplification may either lose important features or retain unnecessary detail [S1].
*   **Boundaries**: The method handles boundaries by ensuring that dividing points on the boundary are processed correctly. The source notes that dividing points on boundaries have fewer shared tetrahedra, which simplifies the local fusion check [S1].
*   **Degenerate Input**: If a tetrahedron has zero volume or coincident vertices, the error calculation may be undefined. The source does not specify handling for this, but standard geometric checks should be applied [G].

## Operating characteristics

*   **Time Complexity**: The complexity is proportional to the number of dividing points and tetrahedra. Since the algorithm visits the dividing point table level by level, it avoids traversing the entire tree structure repeatedly. The source states that visiting a small number of elements reduces rendering time geometrically [S1].
*   **Memory Complexity**: Requires storage for the binary tree array, the dividing point table, and the dividing point structures. The binary tree array stores only vertex indices, saving memory compared to storing explicit child pointers [S1].
*   **Strengths**:
    *   Reduces rendering time significantly by decreasing the number of tetrahedra [S1].
    *   Bottom-up approach ensures that fusion decisions are based on accumulated local errors, leading to a more accurate approximation than top-down methods [S1].
    *   Local fusion does not have a widespread global effect on complexity, making the process efficient [S1].
*   **Limitations**:
    *   Requires pre-computation of errors for all dividing points.
    *   The accuracy depends on the approximation error metric.
    *   Not given in the sources: Performance on highly irregular or noisy data.

## Background

*   **Origin**: The method is part of the Multi-Tetra Framework for visualizing regular volume data [S1].
*   **History**: It builds on the idea of merging tetrahedra to reduce rendering time, as discussed in [S1]. The bottom-up approach is similar to techniques in [LH91] (referenced in the source, but not provided in the citation list) [S1].
*   **Usage**: Used for isosurface extraction and visualization of regular volume data, particularly where real-time or efficient rendering is required [S1].

## Sources

[S1] Multiresolution tetrahedral framework for visualizing regular volume data, https://doi.org/10.5555/266989.267040, fetched 2026-10-30
