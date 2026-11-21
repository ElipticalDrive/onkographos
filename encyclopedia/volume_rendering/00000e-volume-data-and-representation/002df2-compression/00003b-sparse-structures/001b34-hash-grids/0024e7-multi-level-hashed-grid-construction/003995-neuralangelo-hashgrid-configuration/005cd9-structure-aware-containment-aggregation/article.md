# Structure-Aware Containment Aggregation


## Overview
Structure-Aware Containment Aggregation (SACA) is a topology-aware module that modulates coarser-level and finer-level feature aggregations in Hierarchical Anchor Coupling (HAC) by distinguishing anchors participating in valid octree parent-child containment relations from spatially isolated anchors [S1, unofficial]. It reduces the influence of topologically isolated anchors using a binary containment mask or a soft-weighting scheme, complementing the cross-level gradient pathways of HAC [S1, unofficial].

## How it works
SACA operates on the anchor features $\{\hat{f}_i\}_{i \in V}$ enhanced by the level-wise feature adjustment $\delta_l$ from HAC, where $\hat{f}_i = f_i + \delta_{l_i}$ for visible anchors $i$ at level $l_i$ [S1, unofficial]. It computes a containment mask for each anchor to determine its topological validity relative to adjacent levels and applies this mask to weight the aggregation contributions [S1, unofficial].

**Inputs and Outputs**
*   **Inputs**: Anchor positions $\{p_i\}$, anchor levels $\{l_i\}$, octree origin $o$, voxel side lengths $\{v_l\}$ for each level $l$, and the soft-weighting hyperparameter $\alpha$.
*   **Outputs**: A weight factor $w_i$ for each anchor $i$, used to modulate the aggregation operators $A_{par}$ (parent) and $A_{chi}$ (child) [S1, unofficial].

**Data Structures**
*   **Containment Mask ($m_i$)**: A binary value for each anchor $i$. $m_i = 1$ if the anchor has a valid parent or child in the octree hierarchy; $m_i = 0$ if the anchor is topologically isolated (no valid parent or child) [S1, unofficial].

**Steps**
1.  **Grid Coordinate Computation**: For an anchor $a_i$ at level $l_i$ with position $p_i$, compute its grid coordinate at the adjacent coarser level $l_i - 1$ [S1, unofficial]:
    $$ g_{i}^{par} = \text{round}\left(\frac{p_i - o}{v_{l_i - 1}}\right) \in \mathbb{Z}^3 $$
    where $v_{l_i - 1}$ is the voxel side length at level $l_i - 1$ and $o$ is the octree origin [S1, unofficial].
2.  **Hash-Based Matching**: Perform hash-based matching to identify valid parent-child containment relations between anchors at adjacent levels using the computed grid coordinates [S1, unofficial].
3.  **Mask Construction**: Determine the binary containment mask $m_i$ for each anchor. If anchor $i$ participates in a valid containment relation (i.e., a corresponding anchor exists at the parent or child level within the hash grid), set $m_i = 1$. Otherwise, set $m_i = 0$ [S1, unofficial].
4.  **Soft-Weighting Calculation**: Compute the weight $w_i$ using the soft-weighting scheme [S1, unofficial]:
    $$ w_i = m_i + \alpha(1 - m_i) $$
    where $\alpha$ is the soft-weighting hyperparameter [S1, unofficial].
5.  **Aggregation Modulation**: Apply the weight $w_i$ to the aggregation operators $A_{par}$ and $A_{chi}$ to modulate the contribution of anchors in valid relations versus isolated anchors [S1, unofficial].

**Parameters**
*   **$\alpha$**: The soft-weighting hyperparameter. Swept over $\{0, 0.1, 0.2, 0.3, 0.5, 0.7, 1.0\}$. Default $\alpha = 0.3$ across all tested scenes [S1, unofficial].
*   **$v_l$**: Voxel side length at level $l$ [S1, unofficial].
*   **$o$**: Octree origin [S1, unofficial].
*   SACA is parameter-free aside from $\alpha$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_saca_weights(anchors, octree_origin, voxel_side_lengths, alpha):
    """
    Compute SACA weights for anchors.
    
    Args:
        anchors: List of anchor objects with attributes:
            - pos: 3D position vector
            - level: integer level index
        octree_origin: 3D position vector
        voxel_side_lengths: Dictionary mapping level index to voxel side length
        alpha: Float, soft-weighting hyperparameter
        
    Returns:
        weights: List of float weights corresponding to each anchor
    """
    weights = []
    
    # Precompute grid coordinates for parent level matching
    # Note: Hash-based matching details are implementation-specific.
    # This pseudocode assumes a helper function `has_valid_containment`
    # that checks if an anchor at level l has a corresponding anchor
    # at level l-1 or l+1 within the same voxel/containment relation.
    
    for anchor in anchors:
        p = anchor.pos
        l = anchor.level
        
        # Step 1: Compute grid coordinate at coarser level
        # v_l_minus_1 is the voxel side length at level l-1
        # Handle edge case: if l=0, there is no coarser level, 
        # so containment can only be checked against finer level.
        if l > 0:
            v_coarser = voxel_side_lengths[l - 1]
            grid_coord_coarser = round_vector((p - octree_origin) / v_coarser)
        else:
            grid_coord_coarser = None
            
        # Step 2 & 3: Determine binary containment mask
        # Check if anchor has valid parent or child relation
        # This requires access to the global anchor list or hash grid structure
        # to find neighbors at adjacent levels.
        has_valid_parent = False
        has_valid_child = False
        
        if l > 0 and grid_coord_coarser is not None:
            # Check for parent in hash grid at level l-1
            # Implementation detail: Hash grid lookup for grid_coord_coarser
            # at level l-1. If found, has_valid_parent = True.
            has_valid_parent = check_hash_grid_match(grid_coord_coarser, l - 1, anchors)
            
        # Check for child in hash grid at level l+1
        # Similar logic: compute grid coord at finer level and check match
        if l + 1 in voxel_side_lengths:
            v_finer = voxel_side_lengths[l + 1]
            grid_coord_finer = round_vector((p - octree_origin) / v_finer)
            has_valid_child = check_hash_grid_match(grid_coord_finer, l + 1, anchors)
            
        # Binary mask: 1 if valid relation exists, 0 otherwise
        m_i = 1 if (has_valid_parent or has_valid_child) else 0
        
        # Step 4: Soft-weighting
        w_i = m_i + alpha * (1 - m_i)
        
        weights.append(w_i)
        
    return weights

def round_vector(vec):
    """Round each component of a vector to the nearest integer."""
    return [round(x) for x in vec]

def check_hash_grid_match(grid_coord, level, anchors):
    """
    Check if an anchor exists at the specified level and grid coordinate.
    This is a placeholder for the actual hash-based matching logic.
    In practice, this would involve hashing the grid coordinate and level
    to look up in a hash table of anchors at that level.
    """
    # Not given in the sources. Implementation depends on the specific
    # hash grid data structure used in the parent method.
    # Typically: hash_key = hash((grid_coord, level))
    # return hash_key in anchor_hash_table
    return False
```

## Operating characteristics
*   **Time Complexity**: SACA is parameter-free and adds minimal computational overhead. The feature aggregation introduces a small per-view computation, but the lower number of decoded and rasterized Neural Gaussians yields a net improvement in end-to-end rendering speed [S1, unofficial].
*   **Memory Complexity**: SACA adds no parameters (parameter-free) [S1, unofficial].
*   **Performance**: On the Mill19-Building scene, adding the binary containment mask provides an additional +0.56 dB PSNR. Replacing the binary mask with the soft-weighting scheme brings an additional +0.20 dB PSNR [S1, unofficial]. The total improvement of +2.67 dB over the baseline demonstrates that HAC and SACA contribute complementarily [S1, unofficial].
*   **Strengths**: Reduces the influence of topologically isolated anchors, alleviating feature isolation [S1, unofficial]. Adapts to topological sparsity variation across scenes and levels via soft-weighting [S1, unofficial].
*   **Limitations**: Requires hash-based matching to identify containment relations, which may add implementation complexity [S1, unofficial].
*   **When to use**: Applied at the anchor-feature aggregation stage alongside HAC on structured 3DGS backbones like CityGS-X, Octree-GS, and Proxy-GS [S1, unofficial].

## Background
SACA is part of the TopoGS method, introduced in "TopoGS: Topology-Aware Anchor Feature Aggregation for Large-Scale 3D Gaussian Splatting" [S1, unofficial]. It was developed to address feature isolation in large-scale 3D Gaussian Splatting by incorporating octree containment into the aggregation process [S1, unofficial].

## Sources
[S1] TopoGS: Topology-Aware Anchor Feature Aggregation for Large-Scale 3D Gaussian Splatting, https://arxiv.org/abs/2609.27868, fetched 2026-10-30 (unofficial)
