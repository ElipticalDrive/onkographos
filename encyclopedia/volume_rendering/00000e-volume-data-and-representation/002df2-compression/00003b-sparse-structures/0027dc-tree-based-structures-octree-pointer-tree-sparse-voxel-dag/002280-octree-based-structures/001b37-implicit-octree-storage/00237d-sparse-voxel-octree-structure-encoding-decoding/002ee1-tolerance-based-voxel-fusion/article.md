# Tolerance-based Voxel Fusion

Also called: Adaptive Resolution Rendering, Spatial Resolution Coarsening, Spatial domain coarsening, Variable Resolution Rendering

## Overview
Tolerance-based Voxel Fusion is a performance optimization strategy that renders volume data at variable resolutions across the spatial domain by fusing voxels with similar values [S1, S2, unofficial]. It solves the trade-off between rendering fidelity and computational cost by coarsening the representation in regions where visual detail is less critical, achieving significant savings in storage and rendering time without perceptible quality loss in moderate coarsening modes [S1, S2, unofficial]. It differs from image-space skipping, which creates blocky artifacts, by operating in the spatial domain to maintain visual continuity [S1, S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A Sparse Voxel Octree (SVO) structure representing volume data, where each leaf node contains voxel values (typically floating-point scalars) [S1, S2, unofficial].
*   **Input:** A user-specified tolerance threshold $\tau$ [S1, S2, unofficial].
*   **Output:** A modified SVO structure where groups of voxels are fused into single representative nodes, reducing the total node count [S1, S2, unofficial].

**Data Structures**
The method operates on the existing SVO structure defined in [Sparse Voxel Octree Structure Encoding/Decoding](3012-sparse-voxel-octree-structure-encoding-decoding.md). No new data structures are introduced; the optimization modifies the tree topology by merging nodes.

**Algorithm Steps**
1.  Traverse the SVO structure to identify candidate blocks of voxels. A "block" refers to a group of adjacent voxels at the same depth level or a subtree representing a spatial region [S1, S2, unofficial].
2.  For each candidate block, calculate the range of voxel values contained within it. Let $V_{max}$ be the maximum value and $V_{min}$ be the minimum value among all voxels in the block [S1, S2, unofficial].
3.  Compute the difference $\Delta = V_{max} - V_{min}$ [S1, S2, unofficial].
4.  Compare $\Delta$ against the user-specified tolerance $\tau$ [S1, S2, unofficial].
5.  If $\Delta \le \tau$, treat the entire block as identical. Fuse the voxels in this block into a single representative node (or higher-level node) with a value derived from the block (e.g., the mean or one of the values) [S1, S2, unofficial].
6.  If $\Delta > \tau$, retain the original resolution for this block and continue processing its children or adjacent blocks [S1, S2, unofficial].
7.  Repeat until all blocks are processed or the desired coarsening level is reached [S1, S2, unofficial].

**Parameters**
*   **Tolerance ($\tau$):** A scalar threshold determining the maximum allowable variation within a fused block.
    *   *Moderate Coarsening:* $\tau$ is set to achieve ~40% storage saving and ~10% rendering time saving, resulting in images visually indistinguishable from regular rendering [S1, S2, unofficial].
    *   *Aggressive Coarsening:* $\tau$ is set higher to achieve ~90% storage saving and ~30% rendering time saving, suitable for previewing but less visually appealing [S1, S2, unofficial].
    *   *Note:* Exact numerical values for $\tau$ depend on the data range and are not specified in the sources [S1, S2, unofficial].

**Pitfalls and Edge Cases**
*   **Image Space Skipping:** Avoid skipping pixels in image space (rendering at lower screen resolution) as it creates blocky patterns that hamper normal perception of image content [S1, S2, unofficial]. Tolerance-based fusion operates in the spatial domain, avoiding these artifacts [S1, S2, unofficial].
*   **Viewing Position Dependency:** The optimization is based on a fixed viewing position. Changing the viewing position requires re-rendering the entire tree to create a new compositing tree, as the visibility of fused regions may change [S1, S2, unofficial].
*   **Temporal Consistency:** For time-varying data, subtrees representing the back portion of the data may not be completely sampled during front-to-back rendering. An approximation is used where unchanged subtrees between time steps are not re-rendered [S1, S2, unofficial]. To support random viewer movement through the temporal domain, complete trees should be saved at regular intervals [S1, S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tolerance_based_voxel_fusion(svo_root, tolerance):
    """
    Fuses voxels in an SVO if their value range is within tolerance.
    
    Args:
        svo_root: The root node of the Sparse Voxel Octree.
        tolerance: The user-specified tolerance threshold.
        
    Returns:
        The modified SVO root with fused nodes.
    """
    # Helper function to compute min and max values in a subtree
    def get_value_range(node):
        if node.is_leaf():
            return (node.value, node.value)
        
        min_val = float('inf')
        max_val = float('-inf')
        
        for child in node.children:
            if child is not None:
                child_min, child_max = get_value_range(child)
                if child_min < min_val:
                    min_val = child_min
                if child_max > max_val:
                    max_val = child_max
                    
        return (min_val, max_val)

    # Helper function to fuse a subtree into a single node
    def fuse_subtree(node):
        # Create a new leaf node with a representative value
        # The source does not specify the exact representative value calculation.
        # [G] A common approach is to use the mean or the first encountered value.
        # Here we assume a placeholder for the representative value calculation.
        representative_value = compute_representative_value(node)
        new_leaf = LeafNode(value=representative_value)
        return new_leaf

    # Recursive fusion function
    def process_node(node):
        if node.is_leaf():
            return node
        
        # Get the value range of the current node's children
        min_val, max_val = get_value_range(node)
        value_range = max_val - min_val
        
        # Check if the block can be fused
        if value_range <= tolerance:
            # Fuse the entire subtree
            return fuse_subtree(node)
        else:
            # Process children recursively
            for i in range(8):
                if node.children[i] is not None:
                    node.children[i] = process_node(node.children[i])
            return node

    # Start processing from the root
    return process_node(svo_root)

def compute_representative_value(node):
    """
    Computes a representative value for a fused block.
    [G] The source does not specify this function. 
    Implementers should define this (e.g., mean, median, or max).
    """
    # Placeholder implementation: return the minimum value
    # This is a guess [G] as the source is silent on the specific aggregation function.
    return node.value # Simplified for pseudocode completeness
```

## Subtopics

### Adaptive Resolution Rendering
A general term for rendering at different resolutions in different areas. Tolerance-based Voxel Fusion is a specific implementation of this concept using value-based fusion [S1, S2, unofficial].

### Spatial Domain Coarsening
The technique of reducing resolution in the spatial domain rather than image space. This avoids the blocky artifacts associated with image-space skipping [S1, S2, unofficial].

### Variable Resolution Rendering
Rendering with varying levels of detail across the volume. Tolerance-based Voxel Fusion achieves this by fusing voxels based on value similarity [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. The traversal is linear with respect to the number of nodes, but fusion reduces the total node count, potentially speeding up subsequent rendering passes [S1, S2, unofficial].
*   **Memory Complexity:** Significant reduction in storage space. Moderate coarsening achieves ~40% saving; aggressive coarsening achieves ~90% saving [S1, S2, unofficial].
*   **Measured Performance:**
    *   *Moderate Coarsening:* ~10% saving in rendering time; visually indistinguishable from regular rendering [S1, S2, unofficial].
    *   *Aggressive Coarsening:* ~30% saving in rendering time; less visually appealing, suitable for previewing [S1, S2, unofficial].
*   **Strengths:** Dramatic savings in storage and rendering time; maintains visual quality in moderate mode; avoids image-space artifacts [S1, S2, unofficial].
*   **Limitations:** Requires fixed viewing position for optimal performance; changing view requires re-rendering; aggressive coarsening reduces visual appeal [S1, S2, unofficial].
*   **When to Use:** When storage or rendering time is constrained, and the viewer position is relatively static or changes infrequently. Ideal for previewing (aggressive) or final rendering (moderate) [S1, S2, unofficial].
*   **When Not to Use:** When the viewer moves randomly through the temporal domain without pre-saved trees, or when high visual fidelity is required in all regions regardless of value similarity [S1, S2, unofficial].

## Background
Tolerance-based Voxel Fusion was introduced in the context of efficient encoding and rendering of time-varying volume data [S1, S2, unofficial]. It was developed to address the computational costs associated with rendering large, time-varying datasets, such as turbulent vortex flows and thermal convection simulations [S1, S2, unofficial]. The method was tested on datasets including Turbulent Vortex Flow (100 time steps, 128^3 float), Thermal Convection (101 time steps, 128^3 float), Turbulent Jets (150 time steps, 128^3 float), Turbulent Shear Flow (81 time steps, 128^3 float), and Heart Modeling (100 time steps, 128^3 byte) [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
