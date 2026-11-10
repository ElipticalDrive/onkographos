# Tolerance-based Voxel Fusion

Also called: Adaptive Resolution Rendering, Spatial Resolution Coarsening, Spatial domain coarsening, Variable Resolution Rendering

## Overview
Tolerance-based Voxel Fusion is a performance optimization strategy that renders volume data at variable resolutions across the spatial domain by fusing voxels with similar values [S1, unofficial][S2, unofficial]. It solves the trade-off between rendering fidelity and computational cost by coarsening the representation in regions where visual detail is less critical, achieving significant savings in storage and rendering time without perceptible quality loss in moderate coarsening modes [S1, unofficial][S2, unofficial]. It differs from image-space skipping, which creates blocky artifacts, by operating in the spatial domain to maintain visual continuity [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A Sparse Voxel Octree (SVO) structure representing volume data, where each leaf node contains voxel values (typically floating-point scalars) [S1, unofficial][S2, unofficial].
*   **Input:** A user-specified tolerance threshold $\tau$ [S1, unofficial][S2, unofficial].
*   **Output:** A coarsened volume representation where blocks of voxels are treated identically, reducing storage and rendering cost [S1, unofficial][S2, unofficial].

**Data Structures**
The method operates on the existing SVO structure defined in [Sparse Voxel Octree Structure Encoding/Decoding](3012-sparse-voxel-octree-structure-encoding-decoding.md) [S1, unofficial][S2, unofficial]. No new data structures are introduced; the optimization modifies the tree topology by merging nodes [S1, unofficial][S2, unofficial].

**Algorithm Steps**
1. Traverse the SVO structure to identify candidate blocks of voxels [S1, unofficial][S2, unofficial].
2. For each candidate block, determine the maximum ($V_{max}$) and minimum ($V_{min}$) voxel values contained within it [S1, unofficial][S2, unofficial].
3. Compare the difference between $V_{max}$ and $V_{min}$ against the user-specified tolerance $\tau$ [S1, unofficial][S2, unofficial].
4. If the difference is less than or equal to $\tau$, treat the entire block as identical and fuse the voxels into a single representative node [S1, unofficial][S2, unofficial].
5. If the difference exceeds $\tau$, retain the original resolution for this block and continue processing its children or adjacent blocks [S1, unofficial][S2, unofficial].
6. Repeat until all blocks are processed or the desired coarsening level is reached [S1, unofficial][S2, unofficial].

**Parameters**
*   **Tolerance ($\tau$):** A scalar threshold determining the maximum allowable variation within a fused block [S1, unofficial][S2, unofficial].
    *   *Moderate Coarsening:* $\tau$ is set to achieve ~40% storage saving and ~10% rendering time saving, resulting in images visually indistinguishable from regular rendering [S1, unofficial][S2, unofficial].
    *   *Aggressive Coarsening:* $\tau$ is set higher to achieve ~90% storage saving and ~30% rendering time saving, suitable for previewing but less visually appealing [S1, unofficial][S2, unofficial].
    *   *Note:* Exact numerical values for $\tau$ depend on the data range and are not specified in the sources [S1, unofficial][S2, unofficial].

**Pitfalls and Edge Cases**
*   **Image Space Skipping:** Avoid skipping pixels in image space (rendering at lower screen resolution) as it creates blocky patterns that hamper normal perception of image content [S1, unofficial][S2, unofficial]. Tolerance-based fusion operates in the spatial domain, avoiding these artifacts [S1, unofficial][S2, unofficial].
*   **Viewing Position Dependency:** The optimization is based on a fixed viewing position [S1, unofficial][S2, unofficial]. Changing the viewing position requires re-rendering the entire tree to create a new compositing tree, as the visibility of fused regions may change [S1, unofficial][S2, unofficial].
*   **Temporal Consistency:** For time-varying data, subtrees representing the back portion of the data may not be completely sampled during front-to-back rendering [S1, unofficial][S2, unofficial]. An approximation is used where unchanged subtrees between time steps are not re-rendered [S1, unofficial][S2, unofficial]. To support random viewer movement through the temporal domain, complete trees should be saved at regular intervals [S1, unofficial][S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tolerance_based_voxel_fusion(svo_root, tolerance):
    """
    Fuses voxels in an SVO if their value range is within tolerance.
    """
    # Minimal Node structure for self-containment
    class Node:
        def __init__(self, value=None, children=None):
            self.value = value
            self.children = children if children else [None] * 8

    def get_value_range(node):
        if node.value is not None:
            return node.value, node.value
        min_val = float('inf')
        max_val = float('-inf')
        for child in node.children:
            if child is not None:
                c_min, c_max = get_value_range(child)
                if c_min < min_val: min_val = c_min
                if c_max > max_val: max_val = c_max
        return min_val, max_val

    def fuse_subtree(node):
        # [G] The source does not specify how to compute the representative value for a fused block.
        # Implementers should define this (e.g., mean, median, or max).
        # Here we use the minimum value as a standard fallback.
        min_val, _ = get_value_range(node)
        return Node(value=min_val)

    def process_node(node):
        if node.value is not None:
            return node
        min_val, max_val = get_value_range(node)
        if max_val - min_val <= tolerance:
            return fuse_subtree(node)
        else:
            for i in range(8):
                if node.children[i] is not None:
                    node.children[i] = process_node(node.children[i])
            return node

    return process_node(svo_root)
```

## Subtopics

### Adaptive Resolution Rendering
A general term for rendering at different resolutions in different areas [S1, unofficial][S2, unofficial]. Tolerance-based Voxel Fusion is a specific implementation of this concept using value-based fusion [S1, unofficial][S2, unofficial].

### Spatial Domain Coarsening
The technique of reducing resolution in the spatial domain rather than image space [S1, unofficial][S2, unofficial]. This avoids the blocky artifacts associated with image-space skipping [S1, unofficial][S2, unofficial].

### Variable Resolution Rendering
Rendering with varying levels of detail across the volume [S1, unofficial][S2, unofficial]. Tolerance-based Voxel Fusion achieves this by fusing voxels based on value similarity [S1, unofficial][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources [S1, unofficial][S2, unofficial]. The traversal is linear with respect to the number of nodes, but fusion reduces the total node count, potentially speeding up subsequent rendering passes [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Significant reduction in storage space [S1, unofficial][S2, unofficial]. Moderate coarsening achieves ~40% saving; aggressive coarsening achieves ~90% saving [S1, unofficial][S2, unofficial].
*   **Measured Performance:** Moderate Coarsening: ~10% saving in rendering time; visually indistinguishable from regular rendering [S1, unofficial][S2, unofficial]. Aggressive Coarsening: ~30% saving in rendering time; less visually appealing, suitable for previewing [S1, unofficial][S2, unofficial].
*   **Strengths:** Dramatic savings in storage and rendering time; maintains visual quality in moderate mode; avoids image-space artifacts [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires fixed viewing position for optimal performance; changing view requires re-rendering; aggressive coarsening reduces visual appeal [S1, unofficial][S2, unofficial].
*   **When to Use:** When storage or rendering time is constrained, and the viewer position is relatively static or changes infrequently [S1, unofficial][S2, unofficial]. Moderate coarsening is visually indistinguishable from regular rendering, while aggressive coarsening is suitable for previewing [S1, unofficial][S2, unofficial].
*   **When Not to Use:** When the viewer moves randomly through the temporal domain without pre-saved trees, or when high visual fidelity is required in all regions regardless of value similarity [S1, unofficial][S2, unofficial].

## Background
Tolerance-based Voxel Fusion was introduced in the context of efficient encoding and rendering of time-varying volume data [S1, unofficial][S2, unofficial]. It was developed to address the computational costs associated with rendering large, time-varying datasets, such as turbulent vortex flows and thermal convection simulations [S1, unofficial][S2, unofficial]. The method was tested on datasets including Turbulent Vortex Flow (100 time steps, 128^3 float), Thermal Convection (101 time steps, 128^3 float), Turbulent Jets (150 time steps, 128^3 float), Turbulent Shear Flow (81 time steps, 128^3 float), and Heart Modeling (100 time steps, 128^3 byte) [S1, unofficial][S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
