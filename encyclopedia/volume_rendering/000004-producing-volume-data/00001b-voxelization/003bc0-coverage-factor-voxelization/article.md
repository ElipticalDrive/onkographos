# Coverage Factor Voxelization

Also called: 64x Supersampling Coverage, Coverage Factor Computation, Scalar Voxelization

## Overview
Coverage Factor Voxelization computes a scalar-valued volumetric representation where each voxel stores the percentage of its volume covered by a geometric object, rather than a binary inside/outside flag [S1, S2]. It solves the problem of aliasing and partial volume effects in coarse voxel grids by deriving these coverage values from a finer-resolution binary solid voxelization [S1, S2]. It is distinguished from sibling methods like [Interior Voxelize](995-interior-voxelize.md) by providing continuous density estimates via an octree-based aggregation of sub-voxel samples, offering effective 64x supersampling [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** An octree-based binary solid voxelization of a triangle mesh. The octree consists of nodes at multiple levels, where leaf nodes (level-0) contain a sub-grid of voxels (typically $2 \times 2 \times 2$ or similar, resulting in 8 voxels per node, though the text specifies "64x supersampling" implying a specific sub-grid size, likely $4 \times 4 \times 4$ or similar aggregate, but the text explicitly states "fraction of set bits in their voxel sub-grid" and "averaging... children"). The input geometry is an indexed triangle set transformed into voxel space [S1, S2].
*   **Output:** A scalar-valued voxelization where each node stores a coverage factor $C \in [0, 1]$.

**Data Structures:**
*   **Octree Node:** Contains an `inside` flag (boolean) and a list of children (for non-leaf nodes).
*   **Leaf Node (Level-0):** Contains a sub-grid of binary voxels. The size of this sub-grid determines the supersampling factor. The text mentions "64x supersampling" [S1, S2]. This implies the leaf node represents a $4 \times 4 \times 4$ block of voxels (64 voxels) or the aggregation process effectively yields this resolution. The text states coverage is the "fraction of set bits in their voxel sub-grid" [S1, S2].
*   **Coverage Factor:** A scalar value stored in each node, representing the covered volume fraction.

**Steps:**
1.  **Initialize:** Transform input triangle vertices to voxel space. Initialize the voxel grid/octree with zeroes [S1, S2]. Perform binary solid voxelization to populate the leaf-level sub-grids with set bits (1) if covered, unset (0) otherwise [S1, S2].
2.  **Compute Level-0 Coverage:** For each leaf node (level-0), calculate the coverage factor $C_0$ as the fraction of set bits in its voxel sub-grid [S1, S2].
    $$ C_0 = \frac{\text{Number of set bits in sub-grid}}{\text{Total number of voxels in sub-grid}} $$
    If the sub-grid has $N$ voxels, and $k$ are set, $C_0 = k/N$. The text specifies 64x supersampling, suggesting $N=64$ [S1, S2].
3.  **Compute Higher-Level Coverage (Bottom-Up):** For each node at level $i > 0$:
    *   If the node has children, calculate its coverage factor $C_i$ as the average of the coverage factors of its children [S1, S2].
        $$ C_i = \frac{1}{M} \sum_{j=1}^{M} C_{child_j} $$
        where $M$ is the number of children (typically 8 for an octree).
    *   If the node has no children (it is a leaf in the octree structure, but potentially larger than the base voxel grid if the octree depth is limited), its coverage factor is defined by its `inside` flag [S1, S2].
        $$ C_i = \begin{cases} 1.0 & \text{if inside flag is set} \\ 0.0 & \text{otherwise} \end{cases} $$
4.  **Iterate:** Apply step 3 iteratively from the lowest level up to the root node [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_coverage_factors(octree_root):
    """
    Computes coverage factors for all nodes in an octree-based binary solid voxelization.
    Assumes leaf nodes have a sub-grid of voxels.
    """
    # Process nodes in bottom-up order (leaves to root)
    # This requires traversing the tree in post-order or storing nodes by level
    
    for node in post_order_traversal(octree_root):
        if node.is_leaf():
            # Leaf node: coverage is fraction of set bits in sub-grid
            set_bits = count_set_bits(node.sub_grid)
            total_voxels = node.sub_grid.size() # Typically 64 for 64x supersampling
            node.coverage_factor = set_bits / total_voxels
        else:
            # Non-leaf node: coverage is average of children's coverage factors
            if node.children is not None and len(node.children) > 0:
                sum_coverage = sum(child.coverage_factor for child in node.children)
                node.coverage_factor = sum_coverage / len(node.children)
            else:
                # Node has no children: coverage defined by inside flag
                node.coverage_factor = 1.0 if node.inside_flag else 0.0

def count_set_bits(sub_grid):
    """Counts the number of set bits (1s) in the binary sub-grid."""
    count = 0
    for voxel in sub_grid:
        if voxel == 1:
            count += 1
    return count
```

**Parameters:**
*   **Supersampling Factor:** The method offers 64x supersampling [S1, S2]. This implies the leaf sub-grid contains 64 voxels.

**Edge Cases and Pitfalls:**
*   **Precision:** The coverage factor is a fraction. Floating-point precision should be sufficient for $k/64$.
*   **Boundary Conditions:** The `inside` flag for nodes without children must be correctly set during the initial binary voxelization step [S1, S2].
*   **Empty Sub-grids:** If a sub-grid has no set bits, coverage is 0. If all are set, coverage is 1.

## Subtopics

### Interior Voxelize
Computes binary inside/outside flags. Choose when discrete solid representation is needed without density information. Link: [Interior Voxelize](995-interior-voxelize.md).

### Shell-based Voxelization
Creates a shell representation. Choose when surface thickness or boundary representation is critical. Link: [Shell-based Voxelization](1748-shell-based-voxelization.md).

### Velocity Voxelization
Stores velocity vectors. Choose for fluid simulation or motion fields. Link: [Velocity Voxelization](3082-velocity-voxelization.md).

### Anisotropic Voxelization
Handles directional dependencies. Choose for anisotropic materials or directional data. Link: [Anisotropic Voxelization](4331-anisotropic-voxelization.md).

### Box-filtering Density Estimation
Uses box filters for density. Choose when a different smoothing or filtering approach is preferred. Link: [Box-filtering Density Estimation](5987-box-filtering-density-estimation.md).

### Volume Density Stretching
Applies stretching to density. Choose for specific visual effects or deformation. Link: [Volume Density Stretching](8033-volume-density-stretching.md).

### Multi-pattern Volumetric Construction via Voxel Multiplication
Uses multiplication of patterns. Choose for complex pattern-based volumes. Link: [Multi-pattern Volumetric Construction via Voxel Multiplication](8874-multi-pattern-volumetric-construction-via-voxel-multiplicati.md).

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the finest resolution grid, as each voxel is processed once to count bits, and each octree node is processed once to average children.
*   **Memory Complexity:** Requires storage for the octree structure and the coverage factors. The binary voxelization data can be discarded after coverage computation if not needed.
*   **Measured Performance:** Implemented in CUDA using persistent threads for parallel processing of triangles and voxels [S1, S2].
*   **Strengths:** Provides smooth, alias-free density estimates; simple to implement on top of binary voxelization; leverages GPU parallelism.
*   **Limitations:** Requires initial binary voxelization; coverage factors are averages, which may smooth out fine details if the octree depth is insufficient.
*   **When to Use:** When rendering volumetric data with partial volume effects or when smooth density transitions are desired.
*   **When Not to Use:** When binary solid representation is sufficient, or when memory bandwidth is extremely limited and binary flags are preferred.

## Background
Originated in the paper "Fast parallel surface and solid voxelization on GPUs" by Michael Schwarz et al. (SIGGRAPH Asia 2010) [S1, S2]. The method was developed to support applications requiring scalar-valued voxelizations derived from binary solid voxelizations, leveraging the octree structure for efficient computation.

## Sources
[S1] Fast parallel surface and solid voxelization on GPUs, https://doi.org/10.1145/1882261.1866201, fetched 2026-11-03
[S2] PDF Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
