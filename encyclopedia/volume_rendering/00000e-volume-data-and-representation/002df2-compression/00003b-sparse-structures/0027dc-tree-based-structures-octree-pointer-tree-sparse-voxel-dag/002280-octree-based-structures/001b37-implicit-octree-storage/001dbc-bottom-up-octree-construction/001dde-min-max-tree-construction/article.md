# Min/Max Tree Construction

Also called: Acceleration Structure Build, Extrema Tree, GPU Min/Max Tree Construction, GPU Pre-processing, Min/Max Cell Construction, Min/Max Octree Construction, Quasi-interpolation Prefiltering

## Overview
Min/Max Tree Construction builds an acceleration structure embedded within an octree to accelerate volume rendering and isosurface extraction by allowing the traversal algorithm to skip nodes whose scalar value range does not contain the target isovalue [S1][S6, unofficial]. It differs from sibling methods like [Octree Construction from Structured Volumes](1854-octree-construction-from-structured-volumes.md) by specifically computing and storing extrema pairs at interior nodes to enable early culling, rather than relying solely on occupancy or variance thresholds [S3, S6, unofficial].

## How it works
**Inputs:** A scalar 3D volume array and an octree structure built via bottom-up consolidation (see [Bottom-up Octree Construction](1590-bottom-up-octree-construction.md)).
**Outputs:** An octree where interior nodes store min/max scalar pairs, enabling ray traversal or marching cubes to prune subtrees.

**Data Structures:**
The octree nodes are extended to store a `min` and `max` scalar value.
- **Leaf Nodes (Cap Nodes):** Nodes at the maximum depth $d_{max}-1$ that contain 8 scalar voxels. They do *not* store a precomputed min/max pair in the node header to save space; instead, the extrema are computed on-the-fly or derived from their children if they were stored (see Pitfalls).
- **Interior Nodes:** Nodes above $d_{max}-1$. Each stores a `min` and `max` pair representing the bounding range of all scalar values in its subtree.

**Algorithm Steps:**
1. **Identify Cap Nodes:** Locate all nodes at depth $d_{max}-1$. These are the parents of the finest-level scalar voxels [S3, unofficial].
2. **Compute Leaf Extrema (Cap Nodes):** For each cap node, compute the minimum and maximum of the 27 voxels that constitute the cell boundaries. This includes the 8 voxels belonging to the node and their forward neighbors in the positive X, Y, and Z dimensions [S3, S6, unofficial].
   - *Note:* In multiresolution octrees, neighbors may exist at different depths (coarser resolutions). The computation must account for forward neighbors at each depth of the octree to handle cells of any power-of-two width [S3, unofficial].
3. **Propagate Upwards:** For each interior node at depth $d < d_{max}-1$, compute its min/max pair by taking the minimum of its 8 children's `min` values and the maximum of its 8 children's `max` values [S3, unofficial].
   - Formula:
     $$ \text{node.min} = \min_{i=0..7} (\text{child}_i.\text{min}) $$
     $$ \text{node.max} = \max_{i=0..7} (\text{child}_i.\text{max}) $$
4. **Recursive Consolidation:** As child voxels consolidate into a parent during the bottom-up build, remove the child nodes and replace them with the computed parent node containing the aggregated min/max pair [S3, unofficial].
5. **Storage:** Store the min/max pairs in the interior nodes. The leaf nodes (cap nodes) do not store these pairs in the node structure itself to avoid excessive memory overhead [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_min_max_tree(octree, max_depth):
    """
    Computes min/max pairs for all interior nodes of an octree.
    Assumes octree nodes have fields: .min, .max, .children (list of 8 nodes or None).
    """
    # Start from the level just above the finest voxels (cap nodes)
    # Cap nodes are at depth max_depth - 1
    # We process bottom-up, so we start at max_depth - 1 and go up to 0
    
    for depth in range(max_depth - 1, -1, -1):
        nodes_at_depth = get_nodes_at_depth(octree, depth)
        
        for node in nodes_at_depth:
            if node.is_leaf():
                # Cap nodes: compute extrema from the 27 voxels (8 own + forward neighbors)
                # This is done on-the-fly during traversal or precomputed if needed,
                # but NOT stored in the node's min/max fields to save space.
                # [G] Implementation note: If precomputing for cap nodes is desired,
                # fetch the 8 scalar values and their forward neighbors from the 
                # original volume data or neighbor-finding structure.
                pass 
            else:
                # Interior node: aggregate from children
                if node.children is None or len(node.children) == 0:
                    node.min = float('inf')
                    node.max = float('-inf')
                else:
                    child_mins = [child.min for child in node.children if child is not None]
                    child_maxs = [child.max for child in node.children if child is not None]
                    
                    if child_mins:
                        node.min = min(child_mins)
                        node.max = max(child_maxs)
                    else:
                        # Empty node
                        node.min = float('inf')
                        node.max = float('-inf')
```

**Parameters:**
- $d_{max}$: The maximum depth of the octree. The min/max tree storage begins at $d_{max}-1$ [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Memory Footprint:** Storing min/max pairs for every scalar voxel (leaf level) would require over three times the storage of the raw octree data [S3, unofficial]. To avoid this, min/max pairs are omitted at leaf nodes and stored starting at depth $d_{max}-1$ [S3, unofficial].
- **Multiresolution Neighbors:** In multiresolution octrees, a leaf node's cell may extend into regions covered by coarser nodes. The min/max computation for a leaf must include forward neighbors at each depth of the octree, potentially resulting in a looser (less accurate) bounding box than in static-resolution data [S3, unofficial].
- **Neighbor Finding:** Computing leaf extrema requires fast neighbor-finding. Scalar values are retrieved from the original data only for cap nodes; parent nodes are computed solely based on children values [S3, unofficial].

## Subtopics

### [AMR-KDTree Leaf Filtering](2732-amr-kdtree-leaf-filtering.md)
AMR-KDTree Leaf Filtering is a traversal optimization for Adaptive Mesh Refinement (AMR) volume rendering that accelerates isosurface extraction by pruning leaf nodes of an AMR-KDTree based on scalar value ranges. It builds on [Min/Max Tree Construction](1623-min-max-tree-construction.md) by applying the filtering specifically to the leaf nodes of the AMR-KDTree structure, which represent regions where all cells originate from the same brick.

### [Implicit Min/Max KD-Trees](8719-implicit-min-max-kd-trees.md)
Implicit Min/Max KD-Trees are a memory-efficient acceleration structure for coherent iso-surface ray tracing that stores min/max scalar values in a balanced binary kd-tree to enable early subtree culling. They differ from standard kd-trees by eliminating explicit node pointers and split-plane coordinates, relying instead on implicit addressing derived from a balanced tree structure and precomputed level-wise splitting dimensions. This approach significantly reduces memory overhead compared to naive implementations, which require 12 bytes per node (8 bytes for plane/pointers, 4 bytes for min/max), resulting in a 12x input data size overhead.

### Min-Max Grid Preprocessing
Preprocessing step to compute minimum and maximum data values for each block, stored as a 2-channel volume texture. Allows occupancy determination with a single texture lookup. Referenced during the occlusion culling pass where the proxy geometry for occupied regions of the volume is determined. Could be recomputed for smaller volumes on the fly, e.g. when streaming time-varying data sets [S4].

### Min/Max Octree Acceleration
An acceleration structure for volume rendering encoding intensity values with non-zero transparency contributions, effectively encoding a single level of a min/max octree. Differs from standard min/max octree implementations by encoding only a single level. Alternatively, the structure can be an octree of binary values encoding node occupancy, as employed by Levoy [S5, abstract only, unofficial].

### Multiresolution Min/Max Tree Computation
A method to compute bounding min/max pairs for multiresolution voxels, accounting for neighbors at different depths and coarser resolutions to create a looser but accurate acceleration structure. For static-resolution data, compute min/max pairs for the immediate parents of finest voxels (cap nodes), reflecting the min/max of 8 scalar vertices constituting the cell. For multiresolution leaf nodes, compute the minimum and maximum of its cell, accounting for values of neighbors in the positive X and Y dimensions. Include forward-neighbors at each depth of the octree in the min/max computation to handle cells of any power-of-two width [S3, unofficial].

### View-Dependent Frustum Culling with Shearing Transformation
An extension of the min-max octree technique by Livnat and Hansen using a shearing transformation visibility test for view-dependent frustum culling [S3, unofficial].

### Min/Max Tree Acceleration
A technique using an octree built from min/max values of voxel groups to accelerate isosurface extraction by pruning nodes where the isovalue does not fall within the range. Traverse the octree from top-down as a min/max tree. Examine octree nodes only when the desired isovalue falls between the minimum and maximum values stored in the node. Voxels are grouped into eights at the maximum depth level. The min/max tree is built by examining 27 voxels: the original voxels and their neighbors constituting the far boundaries of the cells [S6, unofficial].

## Operating characteristics
- **Memory Footprint:** The min/max tree adds approximately 22% overhead on top of raw scalar octree-compressed data [S3, unofficial]. This is calculated as one min/max pair for 8 voxels (at the cap level) plus pairs for other interior nodes. Storing pairs for all leaves would increase storage by over 3x [S3, unofficial].
- **Compression:** The underlying octree yields 3:1 or 4:1 lossless compression ratios [S3, unofficial].
- **Performance:** GPU pre-processing (quasi-interpolation and min/max cell construction) provides significant speedups. Quasi-interpolation improves performance up to ≈ 19-fold (e.g., MRI-Head CPU 2.412s vs GPU 0.140s, speed-up 17.2). Min/max cell construction improves performance up to ≈ 26-fold (e.g., MRI-Head CPU 95.542s vs GPU 3.555s, speed-up 26.9) [S2].
- **Scaling:** The gain from GPU pre-processing is roughly proportional to the size of the dataset [S2].
- **Comparison:** The octree with min/max acceleration occupies an order of magnitude less memory footprint than a comparable kd-tree or hierarchical grid method [S3, unofficial].

## Background
The min/max tree was first applied to volume rendering by Wilhelms and Van Gelder, who proposed a min-max hierarchy embedded in a branch-on-need octree to trivially ignore empty regions of a volume [S3, S6, unofficial]. Livnat et al. refined the interval tree concept in creating a span space tree [S3, unofficial]. Livnat and Hansen extended the min-max octree technique with view-dependent frustum culling using a shearing transformation visibility test [S3, unofficial]. Westermann et al. implemented adaptive marching cubes on multiresolution octree data [S3, unofficial]. Pesco et al. proposed an occlusion test for the implicit itself, further reducing the cost of isosurface extraction at render-time [S3, unofficial].

## Sources
[S1] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://pdfs.semanticscholar.org/aef9/6e0dd17920d0c70cbf0dd43f494cab1c0bcc.pdf, fetched 2026-10-28
[S2] PDF GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S4] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S5] Fast volume rendering using a shear-warp factorisation ... - ResearchGate, https://www.researchgate.net/publication/243763776_Fast_volume_rendering_using_a_shear-warp_factorisation_of_the_viewing_transform, fetched 2026-10-27 (abstract only, unofficial)
[S6] PDF A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
