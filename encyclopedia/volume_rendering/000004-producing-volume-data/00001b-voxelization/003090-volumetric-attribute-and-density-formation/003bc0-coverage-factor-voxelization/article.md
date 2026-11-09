# Coverage Factor Voxelization

Also called: 64x Supersampling Coverage, Coverage Factor Computation, Scalar Voxelization

## Overview
Coverage Factor Voxelization computes a scalar-valued voxelization where each voxel stores a coverage factor, defined as the percentage of the voxel volume covered by the input object [S1, S2]. It differs from binary solid voxelization methods by providing anti-aliased density estimates derived from a finer-resolution binary representation, effectively offering 64× supersampling within an octree structure [S1, S2]. This method is chosen when smooth density gradients or accurate volume integration are required, as opposed to hard-edged occupancy flags.

## How it works
**Inputs and Outputs**
*   **Input**: An octree-based binary solid voxelization. The octree consists of nodes at various levels. Each node contains an `inside` flag (boolean) and, if it has children, a reference to them. Level-0 nodes correspond to the coarsest grid cells, and their children represent the next finer subdivision (typically $2 \times 2 \times 2$ in an octree, but the method generalizes to any subdivision factor $N$ where $N^3$ is the supersampling factor) [S1, S2].
*   **Output**: A scalar value $C$ for each node in the octree, representing the coverage factor.

**Data Structures**
The method operates on an existing octree structure. Each node $n$ at level $l$ has:
*   `inside`: A boolean indicating if the node is fully inside the solid object.
*   `children`: A list or pointer to child nodes. If `children` is null/empty, the node is a leaf in the context of the subdivision used for coverage calculation.

**Steps**
The coverage factor $C(n)$ for a node $n$ is computed recursively or iteratively from the finest level up to the root.

1.  **Initialize**: For all nodes in the octree, initialize $C(n) = 0.0$.
2.  **Compute Level-0 Coverage**: For each level-0 node (root of a subtree corresponding to a coarse voxel), calculate its coverage factor as the fraction of set bits in its voxel sub-grid. In the context of the cited implementation, this sub-grid corresponds to the finest resolution leaves under that node. If the octree uses a standard binary subdivision per axis (8 children per node), the "64x supersampling" implies a specific depth or a $4 \times 4 \times 4$ subdivision per coarse voxel. However, the text explicitly states "64x supersampling" and "fraction of set bits in their voxel sub-grid". Let $S_0$ be the set of finest-level leaf nodes under a level-0 node. Let $|S_0|$ be the total number of such leaves. Let $I_0$ be the subset of $S_0$ where the `inside` flag is true.
    $$ C(n) = \frac{|I_0|}{|S_0|} $$
    *Note*: The sources specify "64x supersampling". If the subdivision is uniform, $|S_0| = 64$. This implies a $4 \times 4 \times 4$ grid of fine voxels per coarse voxel, or a specific octree depth configuration. The general formula is the ratio of occupied fine voxels to total fine voxels in the sub-grid [S1, S2].
3.  **Compute Higher-Level Coverage**: For any node $n$ at level $l > 0$ (or any non-leaf node in the hierarchy used for coverage), determine $C(n)$ based on its children.
    *   If node $n$ has children $c_1, c_2, \dots, c_k$:
        $$ C(n) = \frac{1}{k} \sum_{i=1}^{k} C(c_i) $$
        This is the average of the coverage factors of its immediate children [S1, S2].
    *   If node $n$ has no children (it is a leaf in the coverage computation hierarchy):
        $$ C(n) = \begin{cases} 1.0 & \text{if } n.\text{inside} \text{ is true} \\ 0.0 & \text{otherwise} \end{cases} $$
        The inside flag defines the coverage factor for leaf nodes [S1, S2].
4.  **Iteration**: Apply step 3 iteratively for all remaining levels, from the finest level up to the root. This ensures that the coverage factor of a parent node accurately reflects the aggregated coverage of its descendants [S1, S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_coverage_factors(octree_root):
    """
    Computes coverage factors for all nodes in the octree.
    Assumes the octree is already built as a binary solid voxelization.
    """
    # Get all nodes in the octree, sorted by depth (finest to coarsest)
    nodes = get_all_nodes_sorted_by_depth_descending(octree_root)
    
    for node in nodes:
        if node.children is None or len(node.children) == 0:
            # Leaf node: coverage is determined by the inside flag
            if node.inside:
                node.coverage = 1.0
            else:
                node.coverage = 0.0
        else:
            # Internal node: coverage is the average of children's coverage
            # Note: Children must have been processed already due to descending order
            child_coverages = [child.coverage for child in node.children]
            if len(child_coverages) > 0:
                node.coverage = sum(child_coverages) / len(child_coverages)
            else:
                # Fallback if children list is empty but not None (edge case)
                node.coverage = 0.0

    return octree_root
```

**Parameters**
*   **Supersampling Factor**: The method offers 64x supersampling [S1, S2]. This corresponds to the ratio of fine voxels to coarse voxels in the sub-grid used for level-0 coverage calculation.

**Edge Cases and Pitfalls**
*   **Precision**: The coverage factor is a floating-point value. Standard `float` precision is sufficient for the averaging operations, but accumulation errors may occur in very deep trees.
*   **Empty Sub-grids**: If a node has no children, the logic relies on the `inside` flag. If the `inside` flag is inconsistent with the actual geometry (e.g., due to errors in the preceding solid voxelization step), the coverage factor will be incorrect.
*   **Non-Uniform Subdivision**: The method assumes a uniform subdivision structure where the averaging is valid. If the octree has irregular branching, the "average" should be weighted by the volume of each child if they represent different volumes, but the sources specify a simple average, implying uniform child volumes [S1, S2].

## Subtopics

### Interior Voxelize
Computes binary occupancy. Use when only hard boundaries are needed. [Interior Voxelize](995-interior-voxelize.md)

### Shell-based Voxelization
Focuses on surface shells. Use when surface thickness is critical. [Shell-based Voxelization](1748-shell-based-voxelization.md)

### Velocity Voxelization
Stores vector attributes. Use for fluid dynamics. [Velocity Voxelization](3082-velocity-voxelization.md)

### Anisotropic Voxelization
Handles directional data. Use for anisotropic materials. [Anisotropic Voxelization](4331-anisotropic-voxelization.md)

### Box-filtering Density Estimation
Uses spatial filtering. Use for smooth density from points. [Box-filtering Density Estimation](5987-box-filtering-density-estimation.md)

### Volume Density Stretching
Adjusts density based on deformation. Use for deformable volumes. [Volume Density Stretching](8033-volume-density-stretching.md)

### Multi-pattern Volumetric Construction via Voxel Multiplication
Combines patterns. Use for complex procedural volumes. [Multi-pattern Volumetric Construction via Voxel Multiplication](8874-multi-pattern-volumetric-construction-via-voxel-multiplicati.md)

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N$ is the number of nodes in the octree. Each node is visited once to compute its coverage factor from its children.
*   **Memory Complexity**: $O(N)$, storing the coverage factor for each node.
*   **Performance**: The method is efficient as it leverages the existing octree structure. The "64x supersampling" is achieved without additional geometric sampling, relying on the pre-computed binary solid voxelization [S1, S2].
*   **Strengths**: Provides smooth, anti-aliased density values; efficient post-processing of solid voxelization; supports variable resolution via octree.
*   **Limitations**: Requires a pre-computed solid voxelization; accuracy depends on the resolution of the underlying binary voxelization; averaging may blur sharp features if the subdivision is too coarse.
*   **When to Use**: When smooth density transitions are needed for volume rendering or physics simulations, and a binary solid voxelization is already available.
*   **When Not to Use**: When hard boundaries are required (use binary voxelization directly) or when the input is not a solid object but a surface mesh requiring direct sampling.

## Background
The method was introduced in the context of fast parallel surface and solid voxelization on GPUs [S1, S2]. It was developed to provide a scalar-valued voxelization output from an octree-based solid voxelization, enabling applications that require density information rather than just occupancy. The approach leverages the hierarchical nature of the octree to compute coverage factors efficiently, offering a 64x supersampling effect relative to the coarse grid [S1, S2].

## Sources
[S1] Fast parallel surface and solid voxelization on GPUs, https://doi.org/10.1145/1882261.1866201, fetched 2026-11-03
[S2] PDF Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
