# Depth Structural Consistency Regularization

Also called: RDSC

## Overview
Depth Structural Consistency Regularization (RDSC) is a geometric regularization term used in NeRF training to enforce continuity constraints on rendered depth maps based on priors from external depth estimation models (e.g., DepthAnythingV2) [S1]. It differs from standard depth supervision by explicitly penalizing structural discontinuities between neighboring pixels in the rendered depth field that contradict the smoothness or boundary structure observed in the reference depth map, rather than just minimizing point-wise depth errors [S1]. It builds on [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Rendered depth values $d^r$ for a set of pixels; Reference depth values $d^{V2}$ from a pre-trained depth estimator (e.g., DepthAnythingV2); Hyperparameter $m_{ar'}$ (small boundary value for minor variations); Hyperparameter $k$ (number of nearest neighbors).
*   **Output:** Scalar regularization loss $\mathcal{L}_{DSC}$ (denoted $R_{DSC}$ in sources).

**Data Structures:**
*   Depth maps are treated as 2D grids of floating-point values.
*   $KNN(\cdot)$ operates on depth values within a local spatial region (small region) to identify structurally similar neighbors.

**Steps:**
1.  For each pixel $k_1$ in the rendered depth map, identify its $k$-nearest neighbors $k_2$ based on depth values within a small spatial region in the reference depth map $d^{V2}$ [S1].
2.  For each pair $(k_1, k_2)$, compute the absolute difference between the rendered depth $d^r_{k_1}$ and the rendered depth of the neighbor $d^r_{k_2}$ [S1].
3.  Subtract the boundary value $m_{ar'}$ from this difference [S1].
4.  Apply the hinge loss (max with 0) to the result [S1].
5.  Sum these values over all pixels $k_1$ and their respective neighbors $k_2$ [S1].

**Formula:**
$$ \mathcal{L}_{DSC} = \sum_{k_1} \sum_{d^{V2}_{k_2} \in KNN(d^{V2}_{k_2})} \max(|d^r_{k_1} - d^r_{k_2}| - m_{ar'}, 0) $$

Where:
*   $k_1$: Index of the current pixel in the rendered depth map.
*   $KNN(d^{V2}_{k_2})$: The set of $k$-nearest neighbors of pixel $k_2$ based on depth values in the reference map $d^{V2}$ within a small region [S1].
*   $d^r_{k_1}, d^r_{k_2}$: Rendered depth values at pixels $k_1$ and $k_2$ respectively [S1].
*   $m_{ar'}$: A small boundary value accounting for minor depth variations between neighboring pixels [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_rdsc_loss(rendered_depth_map, reference_depth_map, k_neighbors, margin):
    """
    Computes the Depth Structural Consistency Regularization loss.
    
    Args:
        rendered_depth_map: 2D array of rendered depth values (d^r).
        reference_depth_map: 2D array of reference depth values (d^V2).
        k_neighbors: Integer, number of nearest neighbors to consider.
        margin: Float, m_ar', the boundary value for minor variations.
        
    Returns:
        Float, the RDSC loss value.
    """
    loss = 0.0
    height, width = rendered_depth_map.shape
    
    # Precompute neighbors for reference map based on depth values in local regions
    # Note: KNN is based on depth values within a small region, not just spatial adjacency.
    # Implementation detail: For each pixel, find k neighbors in a local window 
    # sorted by depth value similarity.
    ref_neighbors = []
    for y in range(height):
        for x in range(width):
            # Define a small local region around (x, y)
            # Extract depth values in this region from reference_depth_map
            local_ref_depths = get_local_depth_values(reference_depth_map, x, y)
            # Find k-nearest neighbors based on depth value similarity
            # This typically involves sorting or distance calculation in depth space
            neighbors_indices = find_k_nearest_neighbors(local_ref_depths, k_neighbors)
            ref_neighbors.append(neighbors_indices)
            
    for y1 in range(height):
        for x1 in range(width):
            k1_idx = (y1, x1)
            # Get neighbors for this pixel from precomputed list
            neighbor_indices = ref_neighbors[k1_idx]
            
            for k2_idx in neighbor_indices:
                y2, x2 = k2_idx
                d_r_k1 = rendered_depth_map[y1, x1]
                d_r_k2 = rendered_depth_map[y2, x2]
                
                diff = abs(d_r_k1 - d_r_k2)
                hinge = max(diff - margin, 0.0)
                loss += hinge
                
    return loss

def get_local_depth_values(depth_map, x, y, window_size=3):
    """Extracts depth values in a local window around (x, y)."""
    h, w = depth_map.shape
    values = []
    for dy in range(-window_size//2, window_size//2 + 1):
        for dx in range(-window_size//2, window_size//2 + 1):
            ny, nx = y + dy, x + dx
            if 0 <= ny < h and 0 <= nx < w:
                values.append(depth_map[ny, nx])
    return values

def find_k_nearest_neighbors(depth_values, k):
    """Finds k nearest neighbors based on depth value similarity."""
    # This is a placeholder for the specific KNN logic.
    # In practice, this might involve comparing each pixel to others in the local region
    # and selecting those with the most similar depth values.
    # Since the source does not specify the exact KNN metric (e.g., Euclidean in depth space),
    # we assume a simple sorting or distance-based selection.
    # For implementation, one would typically compute pairwise distances or ranks.
    # Here, we return indices of the first k elements as a placeholder for the undefined logic.
    # A proper implementation would sort neighbors by depth difference.
    return list(range(min(k, len(depth_values))))
```

**Edge Cases and Pitfalls:**
*   **Precision:** Depth values must be consistent in scale and units between the rendered map and the reference map [S1].
*   **Boundaries:** The "small region" for KNN must be defined carefully to avoid including distant, structurally unrelated pixels [S1].
*   **Degenerate Input:** If the reference depth map has large discontinuities or noise, the KNN selection might be unreliable [S1].

## Subtopics

### [Patch-Based Depth Regularization](10369-patch-based-depth-regularization.md)
Applies bilateral and joint bilateral filters to depth maps to regularize noise while preserving sharp edges and structural details during NeRF training.

## Operating characteristics
*   **Time Complexity:** Depends on the efficiency of the KNN search. For each pixel, a local region is scanned and sorted/compared, making it potentially expensive for large images if not optimized [S1].
*   **Memory Complexity:** Requires storing the rendered depth map and the reference depth map, plus temporary structures for KNN [S1].
*   **Performance:** Enhances overall scene reconstruction quality, particularly for neighboring and distant pixel depths, by clarifying depth information [S1].
*   **Strengths:** Effective in sparse-view scenarios where limited training views lead to loss of structure, by leveraging external depth priors [S1].
*   **Limitations:** Relies on the quality of the external depth model (e.g., DepthAnythingV2) [S1].

## Background
Originated in DASNeRF, which proposes depth consistency optimization, adaptive sampling, and hierarchical structural fusion for sparse view neural radiance fields [S1]. It leverages modern depth estimation models to standardize depth regularization [S1].

## Sources
[S1] DASNeRF: depth consistency optimization, adaptive sampling, and hierarchical structural fusion for sparse view neural radiance fields., https://doi.org/10.1371/journal.pone.0321878, fetched 2026-10-27
