# Octree MIP-mapping

Also called: Octree MIP-map Generation, Voxel Hierarchy Filtering, Writing and Mipmapping Values

## Overview
Octree MIP-mapping is the process of propagating and filtering voxel data from the leaf nodes of a sparse voxel octree up to the interior nodes, creating a multi-resolution hierarchy for efficient ray-grid traversal. It solves the problem of representing high-frequency surface details at fine levels while providing smooth, low-frequency approximations at coarse levels, preventing aliasing during rendering. It differs from regular grid MIP-mapping by explicitly handling the boundary sharing of 3x3x3 voxel bricks in an octree structure, requiring weighted averaging to account for overlapping voxels [S1][S2, unofficial].

## How it works
**Inputs:**
*   An octree structure with $n$ levels, where level 0 is the root and level $n-1$ contains the leaves.
*   Leaf node values, written in a prior step ("Write Leaf Values") [S2, unofficial].
*   Voxel values stored as `RGBA32F` in global memory [S2, unofficial].

**Outputs:**
*   Filtered values for all interior nodes (levels $0$ to $n-2$).

**Data Structures:**
*   **Octree Nodes:** Each node at level $l$ contains a 3x3x3 brick of voxels.
*   **Sub-nodes:** Each node has 8 child nodes (sub-nodes) at level $l+1$.
*   **Voxel Multiplicity:** Due to the 3x3x3 brick layout, boundary voxels are shared between neighboring bricks. A voxel on the boundary of a brick appears in multiple adjacent bricks.

**Parameters:**
*   $n$: Total number of levels in the octree.
*   Weighting Kernel: 3x3x3 Gaussian weighting kernel [S1]. This is considered an optimal reconstruction filter [S1][FP02].

**Steps:**
1.  **Write Leaf Values:** Launch one thread per entry in the voxel-fragment list. Each thread splats and merges voxel-fragment values into the leaf nodes of the octree, using a scheme similar to the regular grid method [S2, unofficial].
2.  **Mipmap Interior Nodes:** Process levels from bottom (level $n-2$) to top (level 0) in $n-1$ steps [S1][S2, unofficial].
    *   At each step, launch one thread per non-empty node of the current level [S2, unofficial].
    *   Each thread computes the filtered value for the current node by averaging the values contained in its eight sub-nodes (children) from the previous level [S1][S2, unofficial].
    *   **Weighting:** Because each node contains a 3x3x3 voxel brick, boundary voxels reappear in neighboring bricks. To prevent double-counting, each voxel must be weighted by the inverse of its multiplicity (the number of bricks it is shared by) [S1].
    *   This weighting results in a 3x3x3 Gaussian weighting kernel [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octree_mipmap(octree, n_levels):
    """
    Perform MIP-mapping on an octree with n_levels.
    Assumes leaf values are already written.
    """
    # Process from level n-2 down to 0
    for level in range(n_levels - 2, -1, -1):
        # Get all non-empty nodes at the current level
        # In practice, nodes are sorted by level in the octree pool [S2, unofficial]
        current_nodes = get_nodes_at_level(octree, level)
        
        # Launch one thread per non-empty node
        for node in current_nodes:
            # Get the 8 sub-nodes (children) at level + 1
            children = get_children(node)
            
            # Initialize accumulator for the 3x3x3 brick
            # Each child is a 3x3x3 brick. We need to combine them.
            # However, the standard description says "averages the values contained in its eight sub-nodes"
            # But it also mentions weighting by inverse multiplicity due to boundary sharing.
            # The "averaging" likely refers to the final result of the weighted sum.
            
            # Let's assume the sub-nodes provide the raw voxel data for the parent's 3x3x3 brick.
            # The parent's brick is composed of the inner parts of the children,
            # but the children's boundary voxels overlap.
            
            # The source [S1] states: "compute the filtered values coming from the bricks in the sub nodes"
            # and "weight each voxel with the inverse of its multiplicity".
            # And "results in a 3x3x3 Gaussian weighting kernel".
            
            # Implementation detail:
            # The parent node's 3x3x3 brick is formed by the 8 children.
            # Each child contributes a 3x3x3 brick.
            # The parent's voxel at position (i,j,k) is a weighted average of the corresponding
            # voxels in the 8 children, but since children overlap, we must handle boundaries.
            
            # A simpler interpretation from [S2, unofficial]: "averages the values contained in its eight sub-nodes"
            # This might mean averaging the *node values* if they are pre-aggregated, but [S1] says "voxels".
            # Let's stick to the voxel-level weighting described in [S1].
            
            # The parent brick is 3x3x3.
            # The children are 3x3x3.
            # The parent's voxel at (x,y,z) is influenced by the children's voxels.
            # The Gaussian kernel implies a weighted sum.
            
            # Since the exact formula for the Gaussian weights is not given, we use a placeholder comment.
            # The source [S1] cites [FP02] for the optimal reconstruction filter.
            
            weighted_sum = 0.0
            weight_total = 0.0
            
            # Iterate over the 3x3x3 voxels of the parent node
            for x in range(3):
                for y in range(3):
                    for z in range(3):
                        # Determine which children contribute to this voxel
                        # And apply the inverse multiplicity weight
                        # The Gaussian kernel is applied here [S1]
                        
                        # Note: The exact mapping of parent voxel to child voxels and the
                        # Gaussian weights are not fully detailed in the provided text.
                        # We assume a standard 3x3x3 Gaussian kernel is applied to the
                        # combined data from the 8 children.
                        
                        # Placeholder for the specific Gaussian weighting logic [FP02]
                        # value = compute_gaussian_weighted_average(children, x, y, z)
                        
                        # For the purpose of this article, we state the operation:
                        # value = weighted_average_of_children_voxels(children, x, y, z)
                        
                        pass
            
            # Set the filtered value for the current node
            # The node value is typically the average of its brick or a representative value
            node.value = compute_node_value(weighted_sum)

def compute_node_value(weighted_sum):
    # Return the aggregated value for the node
    # This might be the mean, or a specific representative value
    return weighted_sum
```

**Edge Cases and Pitfalls:**
*   **Boundary Sharing:** Failure to weight voxels by the inverse of their multiplicity will result in incorrect values at node boundaries due to double-counting [S1].
*   **Precision:** Using `RGBA32F` helps maintain precision during accumulation [S2, unofficial].
*   **Empty Nodes:** Only non-empty nodes need to be processed at each level [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of non-empty nodes in the octree. The process requires $n-1$ passes, one per level [S1][S2, unofficial].
*   **Memory Complexity:** Requires storage for the octree structure and the voxel values. The MIP-mapping itself does not require additional significant memory beyond the existing structure [S1][S2, unofficial].
*   **Performance:** On a Kepler-based NVIDIA GTX680, with a maximum voxelization resolution of $512^3$ (9 octree levels), the MIP-mapping step is relatively fast compared to octree construction [S2, unofficial].
*   **Strengths:** Efficiently creates a multi-resolution hierarchy for ray-grid traversal. Handles sparse data well.
*   **Limitations:** Requires careful handling of boundary voxels. Performance depends on the number of non-empty nodes.

## Background
*   **Origin:** The method is described in the context of interactive indirect illumination using voxel cone tracing [S1] and sparse voxelization for OpenGL [S2, unofficial].
*   **History:** The use of Gaussian weighting kernels for reconstruction is cited from [FP02] [S1]. The specific implementation for sparse voxel octrees is detailed in [S1] and [S2, unofficial].
*   **Usage:** Used in real-time ray tracing and voxel-based rendering techniques [S1][S2, unofficial].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
[S2] K14190_text - Icare3D, https://icare3d.org/research/OpenGLInsights-SparseVoxelization.pdf, fetched 2026-10-29 (unofficial)
