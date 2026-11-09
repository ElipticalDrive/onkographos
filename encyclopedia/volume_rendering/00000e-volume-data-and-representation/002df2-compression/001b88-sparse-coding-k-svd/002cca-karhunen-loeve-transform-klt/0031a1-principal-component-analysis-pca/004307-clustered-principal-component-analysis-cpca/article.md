# Clustered Principal Component Analysis (CPCA)

Also called: CPCA for Visibility, Clustered Principal Component Analysis (CPCA) for Visibility, Clustered Principal Component Analysis (CPCA) for Visibility Compression, Visibility Compression via Clustered Principal Component Analysis (CPCA)

## Overview
Clustered Principal Component Analysis (CPCA) is a compression strategy applied to wavelet coefficients of aggregated interior visibility (AIV) and aggregated boundary visibility (ABV) to reduce memory cost while preserving visual quality, addressing the high memory cost of high-angular-resolution visibility data [S1, unofficial]. It partitions the angular domain into tiles and applies PCA to clusters of tiles covering the same subset of the domain, allowing early culling of visible/occluded tiles [S1, unofficial]. Unlike standard PCA which assumes sample points are on or near a linear subspace globally, CPCA exploits spatial and angular locality by grouping tiles, making it suitable for compressing basis coefficients where global linearity does not hold [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Aggregated visibility data (AIV or ABV) at a specific Level of Detail (LoD). AIV typically uses $32^2$ angular resolution; ABV uses $64^2$ to reconstruct sharp silhouettes [S1, unofficial].
*   **Output:** Compressed representatives for each cluster, stored as wavelet coefficients in an implicit sparse quadtree [S1, unofficial].

**Data Structures:**
1.  **Visibility Matrix ($C$):** For a cluster of tiles, each column is one tile flattened to 1D and subtracted by the cluster mean.
2.  **Implicit Sparse Quadtree:** Stores 2D Haar wavelet coefficients of the representative tiles, supporting random access [S1, unofficial].

**Parameters:**
*   **Block Size:** Individual blocks of no more than $64^3$ voxels [S1, unofficial].
*   **Angular Partitioning:** The domain is partitioned into $4 \times 4$ angular tiles using equal-area mapping [S1, unofficial].
*   **Truncation Rate:** 90% to 95% of wavelet coefficients are truncated after nonlinear approximation [S1, unofficial].
*   **Cluster Configuration:** Example: AIV compressed to 30 clusters each with 10 representatives; ABV compressed to 30 clusters each with 60 representatives [S1, unofficial].
*   **Top-$k$ Vectors:** The number of left-singular vectors retained from $U$ to form representatives [S1, unofficial].

**Steps:**
1.  **Block Division:** Divide the LoD level into individual blocks of no more than $64^3$ voxels. Apply CPCA to each block separately to exploit spatial locality [S1, unofficial].
2.  **Angular Partitioning:** Partition the entire domain into $4 \times 4$ angular tiles using equal-area mapping [S1, unofficial].
3.  **Grouping:** Group tiles that cover the same subset of the domain. Cull tiles that are completely visible or occluded early [S1, unofficial].
4.  **Matrix Construction:** For each group (cluster), construct the visibility matrix $C$ where each column is one tile flattened to 1D and subtracted by the cluster mean [S1, unofficial].
5.  **SVD/Eigendecomposition:** Compute the SVD of $C$:
    $$ C = U \Sigma V^T $$
    To improve efficiency, compute the eigendecomposition of either $C C^T$ or $C^T C$, whichever is smaller. Both produce eigenvalues equal to $\Sigma^2$. The eigenvectors of $C C^T$ are $U$; the eigenvectors of $C^T C$ are $V$. If $V$ is computed, $U$ can be derived via $U = C V \Sigma^{-1}$ [S1, unofficial]. When possible, solve only for the top eigenpairs using the Lanczos algorithm [S1, unofficial].
6.  **Representative Selection:** Take the top $k$ left-singular vectors from the $k$ leftmost columns of $U$ as the representative tiles [S1, unofficial].
7.  **Wavelet Compression:** Compress each representative tile using non-standard 2D Haar wavelet compression [S1, unofficial].
8.  **Storage:** Arrange wavelet coefficients as an implicit sparse quadtree supporting random access [S1, unofficial].

**Rendering Evaluation:**
To evaluate visibility for a given direction, traverse a path down the tree, following a child if its basis supports the direction being evaluated [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cpcp_block(block_voxels, angular_resolution, max_block_voxels=64**3):
    """
    Compress visibility data for a block of voxels using CPCA.
    """
    # Step 1: Block Division (Implicit in input selection)
    # Input: block_voxels (list of voxel visibility data)
    
    # Step 2: Angular Partitioning
    # Partition domain into 4x4 angular tiles using equal-area mapping
    tiles = partition_angular_domain(angular_resolution, grid_size=(4, 4))
    
    # Step 3: Grouping and Culling
    # Group tiles covering the same subset of the domain
    # Cull fully visible/occluded tiles
    clusters = group_tiles_by_domain_subset(tiles, block_voxels)
    
    representatives = []
    
    for cluster in clusters:
        # Step 4: Matrix Construction
        # Flatten tiles to 1D and subtract cluster mean
        C = build_visibility_matrix(cluster.tiles)
        
        # Step 5: SVD/Eigendecomposition
        # Choose smaller matrix for eigendecomposition
        if C.shape[0] < C.shape[1]:
            # Compute eigendecomposition of C @ C.T
            # Use Lanczos for top eigenpairs if possible
            eigenvalues, U = lanczos_eigendecomposition(C @ C.T, k=top_k)
        else:
            # Compute eigendecomposition of C.T @ C to get V
            eigenvalues, V = lanczos_eigendecomposition(C.T @ C, k=top_k)
            # Derive U from V
            # Sigma is sqrt of eigenvalues
            Sigma_inv = 1.0 / np.sqrt(eigenvalues)
            U = (C @ V) * Sigma_inv
            
        # Step 6: Representative Selection
        # Take top k left-singular vectors from U
        reps = U[:, :top_k]
        
        # Step 7 & 8: Wavelet Compression and Storage
        for rep in reps:
            # Compress using non-standard 2D Haar wavelet
            wavelet_coeffs = haar_wavelet_compress(rep)
            # Truncate 90-95% of coefficients
            wavelet_coeffs = truncate_wavelets(wavelet_coeffs, rate=0.95)
            # Store in implicit sparse quadtree
            quadtree = build_sparse_quadtree(wavelet_coeffs)
            representatives.append(quadtree)
            
    return representatives

def evaluate_visibility(quadtree, direction):
    """
    Evaluate visibility for a given direction by traversing the quadtree.
    """
    # Traverse path down the tree
    # Follow child if its basis supports the direction
    node = quadtree.root
    while node.has_children():
        child = node.get_child_supporting_direction(direction)
        if child is None:
            break
        node = child
    return node.value
```

## Operating characteristics
*   **Complexity:** PCA has cubic time complexity and quadratic space complexity with respect to input data matrix size, making it impractical for fine LoD levels directly [S1, unofficial].
*   **Performance:** Achieves an extra ~4x compression ratio with little impact on visual quality in the Colortree scene example [S1, unofficial].
*   **Strengths:** Exploits spatial locality by processing blocks; improves angular locality by partitioning; allows early culling of tiles; supports random access via sparse quadtree during rendering [S1, unofficial].
*   **Limitations:** Cubic/quadratic complexity of PCA limits direct application to fine resolutions; requires heuristic block division [S1, unofficial].
*   **When to use:** When high-angular-resolution visibility data (AIV/ABV) causes high memory costs and global PCA is infeasible due to complexity [S1, unofficial].

## Background
CPCA is a method to compress aggregated visibility (AIV and ABV) using Clustered Principal Component Analysis, similar to Sloan et al. [2003] and Liu et al. [2004] [S1, unofficial]. It is used in efficient scene appearance aggregation for level-of-detail rendering to manage the memory footprint of visibility data [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
