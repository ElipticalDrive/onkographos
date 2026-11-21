# TopoGS

Also called: Topology-Aware Anchor Feature Aggregation for Large-Scale 3D Gaussian Splatting

## Overview
TopoGS is a topology-aware anchor feature aggregation framework designed to address the feature isolation problem in octree-based 3D Gaussian Splatting (3DGS) representations [S1, unofficial]. It solves the issue where anchor features at different octree levels are optimized in isolation, leading to high redundancy and poor cross-level information flow [S1, unofficial]. Unlike standard uniform mean pooling which treats all adjacent-level anchors equally, TopoGS distinguishes anchors participating in valid parent/child containment relations from isolated anchors, establishing bidirectional cross-level gradient pathways [S1, unofficial]. It sets itself apart from sibling methods like [Octree-based Splatting](8067-octree-based-splatting.md) by explicitly modeling the octree's topological structure during feature aggregation rather than relying solely on spatial proximity or error metrics for primitive selection [S1, unofficial].

## How it works
TopoGS operates as a lightweight module situated between the backbone and the decoder of an octree-anchor 3DGS pipeline, leaving the octree structure and rasterization pipeline untouched [S1, unofficial]. It consists of two complementary components: Hierarchical Anchor Coupling (HAC) and Structure-Aware Containment Aggregation (SACA) [S1, unofficial].

### Inputs and Outputs
- **Input:** Anchor features for all levels of the octree hierarchy. Specifically, for a target level $L$, it requires the mean features of level $L$, its adjacent parent level $L-1$, and its adjacent child level $L+1$ [S1, unofficial]. It also requires the spatial coordinates of anchors to determine containment relations [S1, unofficial].
- **Output:** Refined anchor features that incorporate bidirectional cross-level context and structure-aware weighting [S1, unofficial].

### Data Structures
- **Anchor Features:** A tensor containing the learned feature vector for each Gaussian anchor.
- **Octree Hierarchy:** Implicitly defined by the parent-child relationships of the octree nodes.
- **Hash Map:** Used in SACA to map anchor coordinates to their containment relations efficiently [S1, unofficial].

### Algorithm Steps

#### 1. Hierarchical Anchor Coupling (HAC)
HAC establishes bidirectional cross-level gradient pathways by fusing context from adjacent levels [S1, unofficial].

1.  **Context Triplet Construction:** For a target octree level $L$, organize the mean features of the level itself ($F_L$), its adjacent parent level ($F_{L-1}$), and its adjacent child level ($F_{L+1}$) into a context triplet [S1, unofficial].
2.  **Feature Fusion:** Fuse the context triplet through a shared lightweight residual MLP ($\mathcal{M}$) to obtain a level-wise feature increment $\Delta F_L$ [S1, unofficial].
    $$ \Delta F_L = \mathcal{M}(F_{L-1}, F_L, F_{L+1}) $$
    *Note: The specific architecture of $\mathcal{M}$ is described as a "lightweight residual MLP" but exact layer counts or hidden units are not provided in the sources [S1, unofficial].*
3.  **Residual Injection:** Inject the level-wise feature increment residually into every anchor at the target level $L$. If $f_i$ is the original feature of an anchor $i$ at level $L$, the updated feature $f'_i$ is:
    $$ f'_i = f_i + \Delta F_L $$

#### 2. Structure-Aware Containment Aggregation (SACA)
SACA modulates aggregation through octree containment, distinguishing valid parent/child relations from isolated anchors [S1, unofficial].

1.  **Hash Lookup:** For each query anchor, perform a logarithmic-time hash lookup to identify anchors participating in cross-level containment relations [S1, unofficial]. The overall cost for constructing the containment mask is $O(N \log N)$, where $N$ is the number of anchors [S1, unofficial].
2.  **Soft Weighting:** Apply a continuous soft-weighting scheme that interpolates between strict containment and uniform pooling. This accommodates varying topological sparsity across scenes and levels [S1, unofficial].
    *   Let $w_i$ be the weight for anchor $i$. The weighting scheme is not explicitly defined with a formula in the sources, but it is described as a "continuous soft-weighting scheme" [S1, unofficial].
    *   The weights are applied to aggregate features from parent and child anchors based on their containment status [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topo_gs_aggregate(anchors, octree_structure, mlp):
    """
    Performs TopoGS feature aggregation.
    
    Args:
        anchors: List of anchor objects, each with attributes:
                 - level: int (octree level)
                 - feature: Tensor (anchor feature vector)
                 - position: Vector3 (spatial position)
        octree_structure: Object providing methods to find parent/child anchors
                          and check containment.
        mlp: A lightweight residual MLP module.
    
    Returns:
        Updated list of anchors with refined features.
    """
    # Group anchors by level
    levels = group_by_level(anchors)
    
    for level in levels:
        # 1. Hierarchical Anchor Coupling (HAC)
        # Get mean features for current level, parent level, and child level
        mean_current = compute_mean_feature(level)
        mean_parent = compute_mean_feature(level.parent) if level.parent else None
        mean_child = compute_mean_feature(level.child) if level.child else None
        
        # Construct context triplet
        context_triplet = (mean_parent, mean_current, mean_child)
        
        # Fuse context triplet through lightweight residual MLP
        level_increment = mlp(context_triplet)
        
        # Inject increment residually into every anchor at this level
        for anchor in level.anchors:
            anchor.feature = anchor.feature + level_increment
            
    # 2. Structure-Aware Containment Aggregation (SACA)
    # Build containment mask using hash lookup
    containment_mask = build_containment_mask(anchors, octree_structure)
    
    # Apply soft weighting based on containment
    for anchor in anchors:
        # Identify valid parent/child anchors via containment_mask
        valid_neighbors = get_valid_neighbors(anchor, containment_mask)
        
        # Compute soft weights for neighbors
        # Note: Exact formula for soft weighting is not provided in sources [S1, unofficial]
        weights = compute_soft_weights(anchor, valid_neighbors)
        
        # Aggregate features from neighbors using weights
        aggregated_feature = aggregate_features(valid_neighbors, weights)
        
        # Update anchor feature (implementation detail: how aggregated feature is combined
        # with existing feature is not explicitly detailed beyond "modulates aggregation") [S1, unofficial]
        anchor.feature = update_feature(anchor.feature, aggregated_feature)
        
    return anchors

def compute_mean_feature(level_anchors):
    if level_anchors is None or len(level_anchors) == 0:
        return None
    features = [a.feature for a in level_anchors]
    return mean(features)

def build_containment_mask(anchors, octree_structure):
    # Hash-based lookup for containment relations
    # Complexity: O(N log N) [S1, unofficial]
    mask = {}
    for anchor in anchors:
        # Find anchors that are parents or children in the octree hierarchy
        parents = octree_structure.find_parents(anchor)
        children = octree_structure.find_children(anchor)
        mask[anchor.id] = {'parents': parents, 'children': children}
    return mask

def get_valid_neighbors(anchor, containment_mask):
    # Retrieve valid parent and child anchors from the mask
    relations = containment_mask[anchor.id]
    valid_neighbors = []
    for parent_id in relations['parents']:
        valid_neighbors.append(anchors_by_id[parent_id])
    for child_id in relations['children']:
        valid_neighbors.append(anchors_by_id[child_id])
    return valid_neighbors

def compute_soft_weights(anchor, neighbors):
    # Continuous soft-weighting scheme interpolating between strict containment
    # and uniform pooling [S1, unofficial]
    # Exact formula not provided [S1, unofficial]
    # Placeholder for implementation detail
    weights = [1.0 / len(neighbors)] * len(neighbors)
    return weights

def aggregate_features(neighbors, weights):
    aggregated = zeros_like(neighbors[0].feature)
    for neighbor, weight in zip(neighbors, weights):
        aggregated = aggregated + weight * neighbor.feature
    return aggregated

def update_feature(current_feature, aggregated_feature):
    # Implementation detail: How the aggregated feature modifies the current feature
    # is not explicitly detailed in the sources [S1, unofficial]
    # Assuming residual addition or concatenation based on context
    return current_feature + aggregated_feature
```

### Parameters
- **HAC MLP:** A lightweight residual MLP. The specific architecture (layers, hidden units) is not detailed in the sources [S1, unofficial].
- **Soft Weighting Scheme:** A continuous scheme interpolating between strict containment and uniform pooling. The exact interpolation parameter or function is not provided [S1, unofficial].

### Edge Cases and Pitfalls
- **Sparse Topology:** In regions with varying topological sparsity, the soft weighting scheme must adapt to avoid over-aggregating isolated anchors [S1, unofficial].
- **Boundary Conditions:** Levels at the root or leaf of the octree may lack parent or child levels, respectively. The pseudocode handles this by checking for `None` [S1, unofficial].
- **Precision:** The hash lookup for containment relations must be robust to floating-point precision issues in anchor positions [S1, unofficial].

## Subtopics

### Hierarchical Anchor Coupling (HAC)
A module that establishes bidirectional cross-level gradient pathways through per-level context triplets fused by a lightweight residual MLP [S1, unofficial]. Choose this when you need to address feature isolation between adjacent octree levels without altering the topology [S1, unofficial]. See [TopoGS](topogs.md).

### Structure-Aware Containment Aggregation (SACA)
A module that modulates aggregation through octree containment, distinguishing anchors participating in valid parent/child relations from isolated anchors via hash-based matching and applying a soft weighting scheme [S1, unofficial]. Choose this when you need to handle varying topological sparsity across scenes and levels [S1, unofficial]. See [TopoGS](topogs.md).

## Operating characteristics
- **Time Complexity:** The SACA mask construction complexity is $O(N \log N)$, where $N$ is the number of anchors [S1, unofficial]. The HAC module operates at negligible cost [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:**
    - On aerial scenes (Mill19), TopoGS achieves an average PSNR gain of +2.13 dB over the strongest reported PSNR baseline [S1, unofficial].
    - On ground-level scenes (UrbanScene3D), TopoGS achieves an average PSNR gain of +1.78 dB over the strongest reported PSNR baseline [S1, unofficial].
    - On synthetic-cartographic scenes (WHU), TopoGS achieves an average PSNR gain of +0.29 dB over the strongest reported PSNR baseline [S1, unofficial].
    - On Tanks & Temples-Train, TopoGS improves PSNR by +1.82 dB compared to Geo-GS [S1, unofficial].
    - On Mill19-Building, TopoGS improves PSNR by +2.67 dB compared to CityGS-X [S1, unofficial].
    - TopoGS reduces the off-diagonal mean cosine similarity of anchor features across levels from 0.866 (CityGS-X) to 0.108 [S1, unofficial].
- **Strengths:** Reduces feature redundancy, improves PSNR, renders faster, and uses lower memory compared to baselines [S1, unofficial]. Compatible with Octree-GS and Proxy-GS backbones [S1, unofficial].
- **Limitations:** Requires hash-based lookup for containment relations, which may add overhead in highly dynamic scenes [S1, unofficial].
- **When to use:** When using octree-based 3DGS representations for large-scale scenes and experiencing feature isolation or redundancy across levels [S1, unofficial].
- **When not to use:** When the octree structure is not used or when computational resources are extremely limited and the overhead of hash lookups is prohibitive [S1, unofficial].

## Background
TopoGS was proposed in the paper "TopoGS: Topology-Aware Anchor Feature Aggregation for Large-Scale 3D Gaussian Splatting" [S1, unofficial]. It addresses the open question of how to respect the octree's topological structure in 3DGS representations [S1, unofficial]. The method was evaluated on ten scenes across five public benchmarks: Mill19, UrbanScene3D, Tanks & Temples, MatrixCity, and WHU [S1, unofficial]. The code is available at https://github.com/WZ-CS/TopoGS [S1, unofficial].

## Sources
[S1] TopoGS: Topology-Aware Anchor Feature Aggregation for Large-Scale 3D Gaussian Splatting, https://arxiv.org/abs/2609.27868, fetched 2026-10-30 (unofficial)
