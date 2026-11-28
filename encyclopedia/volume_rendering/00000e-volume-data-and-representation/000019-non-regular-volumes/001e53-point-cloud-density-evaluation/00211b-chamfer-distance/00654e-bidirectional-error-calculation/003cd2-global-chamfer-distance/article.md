# Global Chamfer Distance
## Overview
Global Chamfer Distance measures the global geometric accuracy of a reconstructed mesh surface by computing the bidirectional Chamfer Distance between uniformly sampled point sets from the reconstruction and the ground truth [S1]. It is distinguished from its siblings by evaluating the entire surface geometry rather than specific features (like edges) or relative transforms, serving as a standard metric for absolute reconstruction error [S1].

## How it works
**Inputs:**
- $\mathbf{S}_r$: A set of points uniformly sampled from the surface of the reconstructed mesh.
- $\mathbf{S}_{\text{gt}}$: A set of points uniformly sampled from the ground truth surface.

**Outputs:**
- $\text{CD}_{\text{glo}}$: The scalar global Chamfer Distance value.

**Data Structures:**
- Point sets $\mathbf{S}_r$ and $\mathbf{S}_{\text{gt}}$ are stored as arrays of 3D coordinates $(x, y, z)$ [G].
- The cardinality of each set is fixed at 300,000 points [S1].

**Parameters:**
- Number of sampled points: 300,000 per surface [S1].

**Steps:**
1. Sample 300,000 points uniformly from the reconstructed mesh surface to form $\mathbf{S}_r$ [S1].
2. Sample 300,000 points uniformly from the ground truth surface to form $\mathbf{S}_{\text{gt}}$ [S1].
3. Compute the directed Chamfer Distance from $\mathbf{S}_r$ to $\mathbf{S}_{\text{gt}}$:
   $$ d(\mathbf{S}_r \to \mathbf{S}_{\text{gt}}) = \frac{1}{|\mathbf{S}_r|} \sum_{\mathbf{x} \in \mathbf{S}_r} \min_{\mathbf{y} \in \mathbf{S}_{\text{gt}}} \|\mathbf{x} - \mathbf{y}\|_2 $$
4. Compute the directed Chamfer Distance from $\mathbf{S}_{\text{gt}}$ to $\mathbf{S}_r$:
   $$ d(\mathbf{S}_{\text{gt}} \to \mathbf{S}_r) = \frac{1}{|\mathbf{S}_{\text{gt}}|} \sum_{\mathbf{y} \in \mathbf{S}_{\text{gt}}} \min_{\mathbf{x} \in \mathbf{S}_r} \|\mathbf{y} - \mathbf{x}\|_2 $$
5. Sum the two directed distances to obtain the symmetric Global Chamfer Distance:
   $$ \text{CD}_{\text{glo}} = d(\mathbf{S}_r \to \mathbf{S}_{\text{gt}}) + d(\mathbf{S}_{\text{gt}} \to \mathbf{S}_r) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def global_chamfer_distance(points_r: list, points_gt: list) -> float:
    """
    Computes the Global Chamfer Distance between two point sets.
    
    Args:
        points_r: List of (x, y, z) tuples representing points from the reconstructed mesh.
        points_gt: List of (x, y, z) tuples representing points from the ground truth.
        
    Returns:
        The symmetric Chamfer Distance.
    """
    n = len(points_r)
    m = len(points_gt)
    
    # Compute directed distance from r to gt
    sum_r_to_gt = 0.0
    for p in points_r:
        min_dist_sq = float('inf')
        for q in points_gt:
            dx = p[0] - q[0]
            dy = p[1] - q[1]
            dz = p[2] - q[2]
            dist_sq = dx*dx + dy*dy + dz*dz
            if dist_sq < min_dist_sq:
                min_dist_sq = dist_sq
        sum_r_to_gt += math.sqrt(min_dist_sq)
    mean_r_to_gt = sum_r_to_gt / n
    
    # Compute directed distance from gt to r
    sum_gt_to_r = 0.0
    for q in points_gt:
        min_dist_sq = float('inf')
        for p in points_r:
            dx = p[0] - q[0]
            dy = p[1] - q[1]
            dz = p[2] - q[2]
            dist_sq = dx*dx + dy*dy + dz*dz
            if dist_sq < min_dist_sq:
                min_dist_sq = dist_sq
        sum_gt_to_r += math.sqrt(min_dist_sq)
    mean_gt_to_r = sum_gt_to_r / m
    
    return mean_r_to_gt + mean_gt_to_r
```

**Edge Cases and Pitfalls:**
- **Sampling Uniformity:** The metric assumes uniform sampling of the underlying surfaces to ensure statistical validity [S1].
- **Degenerate Input:** If either surface is empty, the distance is undefined [G].

## Subtopics

### [Geometric Consistency Metric](16683-geometric-consistency-metric.md)
A scale-normalized consistency measure used to evaluate geometric consistency between reconstructed and reference surfaces, serving as a measure of shape fidelity rather than an absolute metrological error. It differs from Global Chamfer Distance by normalizing the bidirectional nearest-surface distances by the reference bounding box diagonal, making it robust to scale variations and suitable for comparing objects of different sizes.

## Operating characteristics
- **Measured Performance:** Computed using 300K sampled points per surface [S1].
- **Strengths:** Provides a single scalar value for global geometric accuracy [S1].
- **Limitations:** Does not account for local geometric features like edges or boundaries specifically (unlike $CD_{nme}$ or $CD_{bde}$) [S1].
- **When to use:** For evaluating overall surface reconstruction quality against a ground truth mesh [S1].

## Background
Global Chamfer Distance was introduced as a metric for evaluating the geometric accuracy of reconstructed meshes, specifically in the context of neural unsigned distance functions (UDFs) and zero-level set extraction [S1]. It is derived from the general definition of Chamfer Distance, which measures the bidirectional distance between two point sets [S1]. The metric is part of a family of Chamfer-based metrics that also includes variants for non-manifold edges ($CD_{nme}$) and boundary edges ($CD_{bde}$) [S1].

## Sources
[S1] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
