# Depth Structural Consistency Regularization
Also called: RDSC
## Overview
Depth Structural Consistency Regularization (RDSC) enforces geometric continuity in Neural Radiance Fields (NeRF) by aligning the structural relationships of rendered depth pixels with those of a prior depth map [S1]. Unlike direct pixel-wise matching, it constrains the relative depth differences between neighboring pixels to match the continuity observed in a source such as DepthAnythingV2, thereby preserving sharp edges while smoothing noise [S1].
## How it works
**Inputs and outputs:**
*   Input: Rendered expected depth map $\hat{D}$, prior depth map $D_{\text{prior}}$ (e.g., from DepthAnythingV2), hyperparameter $m_{ar'}$ (boundary value for minor variations), and neighborhood size $k$ for KNN.
*   Output: Scalar regularization loss $\mathcal{L}_{\text{RDSC}}$.

**Data structures:**
*   $\hat{D}$: 2D array of floats, shape $(H, W)$, containing $\hat{D}(\mathbf{r})$ for each pixel.
*   $D_{\text{prior}}$: 2D array of floats, shape $(H, W)$, containing depth values from the prior model.

**Steps:**
1.  For each pixel $k_1$ in the rendered depth map $\hat{D}$:
2.  Identify the corresponding pixel location in $D_{\text{prior}}$.
3.  Find the $k$-nearest neighbors ($KNN$) of the depth value $D_{\text{prior}}[k_1]$ within a small spatial region in $D_{\text{prior}}$. Let this set be $\mathcal{N}_{k_1}$.
4.  For each neighbor $k_2 \in \mathcal{N}_{k_1}$:
    *   Retrieve the rendered depth value $d^{r}_{k_1} = \hat{D}[k_1]$ and $d^{r}_{k_2} = \hat{D}[k_2]$.
    *   Compute the absolute difference $|d^{r}_{k_1} - d^{r}_{k_2}|$.
    *   Apply the hinge-like penalty: $\max(|d^{r}_{k_1} - d^{r}_{k_2}| - m_{ar'}, 0)$.
5.  Sum the penalties over all pixels $k_1$ and their neighbors $k_2$.

**Formula:**
$$ \mathcal{L}_{\text{RDSC}} = \sum_{k_1} \sum_{d^{r}_{k_2} \in KNN(d^{r}_{k_2})} \max(|d^{r}_{k_1} - d^{r}_{k_2}| - m_{ar'}, 0) $$

Where:
*   $k_1$ indexes pixels in the rendered depth map.
*   $KNN(\cdot)$ returns the set of $k$-nearest neighbors based on depth values within a small region in the prior map.
*   $d^{r}_{k_1}$ and $d^{r}_{k_2}$ are the rendered depth values at pixels $k_1$ and $k_2$.
*   $m_{ar'}$ is a small boundary value accounting for minor depth variations between neighboring pixels [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_rdsc_loss(rendered_depth, prior_depth, k_neighbors=5, margin=0.01):
    """
    Compute Depth Structural Consistency Regularization loss.
    
    Args:
        rendered_depth: 2D array (H, W) of float, rendered expected depth.
        prior_depth: 2D array (H, W) of float, prior depth map (e.g., DepthAnythingV2).
        k_neighbors: int, number of nearest neighbors to consider.
        margin: float, boundary value for minor depth variations (m_ar').
        
    Returns:
        float, the RDSC loss value.
    """
    H, W = rendered_depth.shape
    total_loss = 0.0
    
    # Precompute flattened arrays for easier indexing
    flat_rendered = rendered_depth.flatten()
    flat_prior = prior_depth.flatten()
    num_pixels = H * W
    
    # For each pixel k1
    for k1 in range(num_pixels):
        # Get the prior depth value at k1
        d_prior_k1 = flat_prior[k1]
        
        # Find k-nearest neighbors in the prior depth map based on depth value similarity
        # Note: The source specifies KNN based on depth values within a small region.
        # Implementation detail: We consider spatial neighbors first, then sort by depth difference.
        # A robust implementation would define a local window (e.g., 3x3 or 5x5) around k1.
        
        # Get spatial neighbors of k1 (excluding k1 itself)
        y1, x1 = divmod(k1, W)
        neighbors_indices = []
        for dy in [-1, 0, 1]:
            for dx in [-1, 0, 1]:
                if dy == 0 and dx == 0:
                    continue
                ny, nx = y1 + dy, x1 + dx
                if 0 <= ny < H and 0 <= nx < W:
                    neighbors_indices.append(ny * W + nx)
        
        # Sort neighbors by their depth value difference to d_prior_k1
        # This approximates "KNN based on depth values within a small region"
        neighbor_depths = [(abs(flat_prior[n] - d_prior_k1), n) for n in neighbors_indices]
        neighbor_depths.sort(key=lambda x: x[0])
        
        # Select top k neighbors
        selected_neighbors = [n for _, n in neighbor_depths[:k_neighbors]]
        
        # Compute loss for these neighbors
        d_r_k1 = flat_rendered[k1]
        for k2 in selected_neighbors:
            d_r_k2 = flat_rendered[k2]
            diff = abs(d_r_k1 - d_r_k2)
            if diff > margin:
                total_loss += (diff - margin)
                
    return total_loss
```

**Parameters:**
*   $m_{ar'}$: Small boundary value. Not given in the sources. [G] Typical values might be small positive floats (e.g., 0.01) to allow minor variations.
*   $k$: Number of neighbors. Not given in the sources. [G] Common values are 3, 5, or 9.

**Edge cases and pitfalls:**
*   If the prior depth map has missing values (NaNs), the KNN search may fail. Handle by skipping pixels with NaN in the prior map.
*   The definition of "small region" for KNN is not explicitly defined in the source. [G] A local spatial window (e.g., 3x3) is a reasonable implementation assumption.
*   Precision issues may arise if depth values vary significantly in scale. Normalization may be required.
## Subtopics

### [Patch-Based Depth Regularization](10369-patch-based-depth-regularization.md)
Uses bilateral filters for edge-preserving smoothing. Choose when explicit edge preservation is needed without KNN complexity.

## Operating characteristics
*   Time complexity: $O(H \cdot W \cdot k)$, where $H, W$ are image dimensions and $k$ is the number of neighbors.
*   Memory complexity: $O(H \cdot W)$ for storing depth maps.
*   Strengths: Enforces structural consistency, handles minor variations via margin, improves sparse-view reconstruction quality [S1].
*   Limitations: Depends on the quality of the prior depth map; KNN search can be computationally expensive if not optimized.
*   When to use: Sparse-view NeRF training where depth continuity is critical and a reliable prior depth map is available [S1].
## Background
Originated in DASNeRF, which combines depth consistency optimization, adaptive sampling, and hierarchical structural fusion for sparse-view neural radiance fields [S1]. It leverages modern depth estimation models like DepthAnythingV2 to provide robust depth priors [S1].
## Sources
[S1] DASNeRF: depth consistency optimization, adaptive sampling, and hierarchical structural fusion for sparse view neural radiance fields., https://doi.org/10.1371/journal.pone.0321878, fetched 2026-10-27
