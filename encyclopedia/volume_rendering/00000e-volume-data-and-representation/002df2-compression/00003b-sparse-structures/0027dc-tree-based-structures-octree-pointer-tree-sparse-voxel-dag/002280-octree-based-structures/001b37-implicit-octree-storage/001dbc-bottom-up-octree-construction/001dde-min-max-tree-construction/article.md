# Min/Max Tree Construction

Also called: Acceleration Structure Build, Extrema Tree, GPU Min/Max Tree Construction, GPU Pre-processing, Min/Max Cell Construction, Min/Max Octree Construction, Quasi-interpolation Prefiltering

## Overview
Min/Max Tree Construction builds a hierarchical acceleration structure embedded within an octree to accelerate ray traversal by storing the minimum and maximum scalar values for each node [S1][S3, unofficial]. It solves the problem of inefficient traversal in sparse or large-volume rendering by allowing the renderer to reject entire subtrees when the target isovalue falls outside the node's computed extrema [S1][S3, unofficial]. Unlike sibling methods that focus on structural compression or partitioning, this method specifically optimizes traversal speed by embedding scalar bounds directly into the tree topology, enabling early-out checks during ray casting [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar voxel array (raw volume data) and an octree structure built via bottom-up consolidation [S1][S3, unofficial].
*   **Output:** An augmented octree where interior nodes store min/max pairs, serving as an acceleration structure for ray tracing [S1][S3, unofficial].

**Data Structures:**
The min/max tree is stored within the same octree structure used for voxel data [S1].
*   **Leaf Nodes:** Do not store min/max pairs explicitly to save space. Bounds are computed on-the-fly from the 8 constituent voxels and their forward neighbors [S3, unofficial].
*   **Interior Nodes (Depth $d_{max}-1$ and above):** Store a min/max pair $(min, max)$ for each scalar field [S3, unofficial].
*   **Cap Nodes:** Nodes at the maximal depth of the octree that contain 8 scalar values (voxels) [S3, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize the octree structure and raw voxel data [S3, unofficial].
2.  **Cap Node Processing:** Retrieve scalar values from the original data only for cap nodes [S3, unofficial].
3.  **Leaf Bounds Computation:** For leaf nodes, compute the minimum and maximum of the 27 voxel values associated with the cell corner [S3, unofficial]. This includes the 8 voxels of the cell and their forward neighbors to handle boundary conditions and neighbor-finding [S3, unofficial].
4.  **Bottom-Up Propagation:** Starting at depth $d_{max}-1$, compute the min/max pair for each parent node by taking the minimum of all children's minimums and the maximum of all children's maximums [S3, unofficial].
    *   Formula for node $P$ with children $C_0 \dots C_7$:
        $$ P.min = \min_{i=0}^{7} (C_i.min) $$
        $$ P.max = \max_{i=0}^{7} (C_i.max) $$
5.  **Recursive Consolidation:** When child voxels consolidate into a parent node during the octree build, remove the corresponding child nodes and update the parent's min/max based on the remaining children [S3, unofficial]. Repeat recursively until the root node is completed [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_min_max_tree(octree, raw_voxels, d_max):
    """
    Computes min/max bounds for the octree bottom-up.
    octree: List of lists where octree[depth] contains all nodes at that depth.
    raw_voxels: 3D scalar array (numpy-like or standard 3D list).
    d_max: Maximum depth of the octree.
    """
    # Process from depth d_max - 1 up to root (depth 0)
    for depth in range(d_max - 1, -1, -1):
        nodes_at_depth = octree[depth]
        for node in nodes_at_depth:
            if node.is_leaf():
                # Leaf nodes do not store min/max pairs explicitly.
                # Bounds are computed on-the-fly during traversal.
                node.min = None
                node.max = None
            else:
                # Interior nodes compute bounds from children
                children = node.children
                if not children:
                    continue
                
                # Initialize with extreme values
                min_val = float('inf')
                max_val = float('-inf')
                
                for child in children:
                    # If child is a leaf, compute its bounds from raw data
                    if child.min is None:
                        voxel_bounds = compute_leaf_bounds(child, raw_voxels)
                        min_val = min(min_val, voxel_bounds[0])
                        max_val = max(max_val, voxel_bounds[1])
                    else:
                        min_val = min(min_val, child.min)
                        max_val = max(max_val, child.max)
                
                node.min = min_val
                node.max = max_val

def compute_leaf_bounds(node, raw_voxels):
    """
    Computes min/max for a leaf node based on 27 voxels (cell + forward neighbors) [S3, unofficial].
    """
    # Get the 8 voxels of the leaf cell
    voxels = raw_voxels[node.x:node.x+8, node.y:node.y+8, node.z:node.z+8]
    
    # Get forward neighbors (positive X, Y, Z directions) to handle 
    # cell corner reconstruction [S3, unofficial]
    neighbors = raw_voxels[node.x+1:node.x+9, node.y+1:node.y+9, node.z+1:node.z+9]
    
    all_values = voxels.flatten()
    neighbors_vals = neighbors.flatten()
    
    return (min(all_values + neighbors_vals), max(all_values + neighbors_vals))
```

**Edge Cases and Pitfalls:**
*   **Storage Overhead:** Storing min/max pairs for every scalar voxel demands over three times the storage of the raw octree data [S3, unofficial]. To avoid this, min/max pairs are omitted at leaf nodes and stored starting at depth $d_{max}-1$ [S3, unofficial].
*   **Neighbor Finding:** Leaf nodes must account for forward-adjacent voxels to correctly compute cell corner extrema, requiring efficient neighbor-finding routines [S3, unofficial].
*   **Data Updates:** If the volume data changes, the min/max tree must be recomputed or renderings will be incorrect [G].

## Subtopics

### [AMR-KDTree Leaf Filtering](2732-amr-kdtree-leaf-filtering.md)
AMR-KDTree Leaf Filtering is a traversal optimization for Adaptive Mesh Refinement (AMR) volume rendering that accelerates isosurface extraction by pruning leaf nodes of an AMR-KDTree based on scalar value ranges. It builds on Min/Max Tree Construction by applying the filtering specifically to the leaf nodes of the AMR-KDTree structure, which represent regions where all cells originate from the same brick. Unlike sibling methods that may filter at coarser levels or use complex bounding volumes, this method leverages the pre-computed min/max values of the finest-level cells within each leaf to immediately discard regions that do not contain the target isovalue, reducing redundant octant generation and vertex computation.

### Min-Max Grid Preprocessing
Min-Max Grid Preprocessing is a preprocessing step that computes minimum and maximum data values for each block of a volume, stored as a 2-channel volume texture [S4]. This allows occupancy determination with a single texture lookup during the occlusion culling pass, where proxy geometry for occupied regions is determined [S4]. It can be recomputed on the fly for smaller volumes, such as when streaming time-varying data sets [S4].

### Min/Max Octree Acceleration
Min/Max Octree Acceleration is an acceleration structure for volume rendering encoding intensity values with non-zero transparency contributions, effectively encoding a single level of a min/max octree [S5, abstract only, unofficial]. It differs from standard min/max octree implementations by encoding only a single level [S5, abstract only, unofficial]. Alternatively, the structure can be an octree of binary values encoding node occupancy, as employed by Levoy [S5, abstract only, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The min/max tree adds approximately 22% additional footprint on top of raw scalar octree-compressed data, calculated as one min/max pair for 8 voxels plus pairs for other interior nodes [S3, unofficial]. The octree itself yields 3:1 or 4:1 lossless compression ratios [S3, unofficial].
*   **Time:** Construction is performed bottom-up during octree consolidation [S1]. GPU pre-processing (including min/max cell construction) can achieve speed-ups of up to ≈ 26-fold compared to CPU (e.g., 95.542s vs 3.555s for MRI-Head) [S2]. The gain from GPU pre-processing is roughly proportional to the size of the dataset [S2].

**Performance and Scaling:**
*   **Acceleration:** The structure allows rejecting children without traversing them, sparing cache misses [S3, unofficial]. It occupies an order of magnitude less memory footprint than a comparable kd-tree or hierarchical grid method [S3, unofficial].
*   **Strengths:** Significant reduction in traversal time for ray casting [S1][S3, unofficial]. Effective for large datasets where cache misses are a bottleneck [S3, unofficial].
*   **Limitations:** The 22% memory overhead is non-trivial [S3, unofficial]. Requires recomputation if data changes [G].

**When to Use:**
*   Use when rendering large sparse volumes with ray tracing where isovalue ranges are known or constrained [S1][S3, unofficial].
*   Use when memory bandwidth is a bottleneck and cache misses from traversal are significant [S3, unofficial].

**When Not to Use:**
*   Use when memory is extremely constrained and the 22% overhead is prohibitive [S3, unofficial].
*   Use when the volume data is static and small enough that simple traversal is sufficient [G].

## Background
The min/max tree was introduced as an acceleration structure for ray tracing implicit surfaces in interactive applications [S3, unofficial]. It was developed to allow efficient traversal of octree volumes by embedding scalar bounds directly into the tree structure [S3, unofficial]. The concept is also present in VolPack, where it is used to accelerate rendering unclassified volumes and computing classified volumes [S2]. GPU implementations have been developed to accelerate the pre-processing stages, such as quasi-interpolation and min/max cell construction, leveraging OpenCL for significant performance gains [S2].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://pdfs.semanticscholar.org/aef9/6e0dd17920d0c70cbf0dd43f494cab1c0bcc.pdf, fetched 2026-10-28
[S2] GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S4] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S5] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
