# Isosurface Similarity Map (ISM)
Also called: SMV
## Overview
The Isosurface Similarity Map (ISM) is a method for analyzing the structural relationships within a volume dataset by measuring the similarity between all pairs of isosurfaces generated from a discrete set of isovalues. Unlike traditional histograms that visualize the frequency of individual scalar values, the ISM provides a global overview of how the geometry of the volume changes across different thresholds by computing the normalized mutual information between the distance fields of corresponding isosurfaces. It is distinct from its sibling [Wasserstein Distance](7206-wasserstein-distance.md) in that it relies on information-theoretic measures of distribution overlap rather than optimal transport costs.
## How it works
**Inputs:**
*   A scalar volume dataset $V$.
*   A discrete set of $N$ isovalues $H = \{h_1, h_2, \dots, h_N\}$.
*   Parameters for distance field computation and histogram binning (bin width, range).

**Outputs:**
*   An $N \times N$ symmetric matrix $SMV$, where $SMV(i, j)$ represents the similarity between the isosurface defined by $h_i$ and the isosurface defined by $h_j$.
*   Optionally, the Isosurface Similarity Distribution (SDV) vector.

**Data Structures:**
*   **Distance Fields:** For each isosurface $h_k$, a signed distance field $D_k(x)$ is computed for every voxel position $x$ in the volume.
*   **Joint Histogram:** A 2D histogram $H_{joint}$ with dimensions $B_x \times B_y$, where bins represent ranges of distance values.
*   **Marginal Histograms:** 1D histograms $H_X$ and $H_Y$ derived from the joint histogram.

**Steps:**

1.  **Generate Isosurfaces and Distance Fields:**
    For each isovalue $h_k \in H$:
    a. Extract the isosurface $S_k$ where the scalar value equals $h_k$.
    b. Compute the signed distance field $D_k(x)$ for all voxels $x$. The sign is typically positive outside and negative inside the isosurface (or vice versa, consistently). [S1]

2.  **Compute Pairwise Similarity for all $(i, j)$ pairs:**
    For each pair of indices $(i, j)$ where $1 \le i, j \le N$:
    a. Let $D_p = D_i$ and $D_q = D_j$ be the distance fields for the two isosurfaces.
    b. Initialize a 2D joint histogram $H_{joint}$ of size $B \times B$ (or appropriate dimensions based on distance ranges) to zero. [S1]
    c. Iterate over every voxel position $x$ in the volume:
       i. Let $val_p = D_p(x)$ and $val_q = D_q(x)$.
       ii. Determine the bin indices $b_x$ and $b_y$ corresponding to $val_p$ and $val_q$ based on the predefined bin ranges.
       iii. Increment $H_{joint}(b_x, b_y)$ by 1. [S1]
    d. Estimate the marginal probability distributions:
       i. Compute $P_X(b_x) = \sum_{b_y} H_{joint}(b_x, b_y) / TotalVoxels$.
       ii. Compute $P_Y(b_y) = \sum_{b_x} H_{joint}(b_x, b_y) / TotalVoxels$.
       iii. Compute the joint probability $P_{XY}(b_x, b_y) = H_{joint}(b_x, b_y) / TotalVoxels$. [S1]
    e. Compute the entropies:
       i. $H(X) = -\sum_{b_x} P_X(b_x) \log_2 P_X(b_x)$ (handling $0 \log 0 = 0$).
       ii. $H(Y) = -\sum_{b_y} P_Y(b_y) \log_2 P_Y(b_y)$.
       iii. $H(X, Y) = -\sum_{b_x, b_y} P_{XY}(b_x, b_y) \log_2 P_{XY}(b_x, b_y)$. [S1]
    f. Compute the Normalized Mutual Information (NMI) as the similarity measure $SMV(i, j)$:
       $$ SMV(i, j) = \frac{2 \cdot (H(X) + H(Y) - H(X, Y))}{H(X) + H(Y)} $$
       *Note: The source cites "Equation 7" for normalized mutual information. Standard NMI definitions vary. The formula above is a common symmetric normalization. If the source implies a different normalization, the implementer must verify Equation 7 from [S1]. The property that the diagonal is 1 and the matrix is symmetric holds for any valid NMI definition.* [S1]

3.  **Construct the Map:**
    Populate the $N \times N$ matrix $SMV$ with the computed values. The matrix is symmetric ($SMV(i, j) = SMV(j, i)$) and has ones along the main diagonal ($SMV(i, i) = 1$). [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_isosurface_similarity_map(volume, isovalues, num_bins=256, dist_range=(-10.0, 10.0)):
    """
    Computes the Isosurface Similarity Map (ISM) for a volume.
    
    Args:
        volume: 3D numpy array of scalar values.
        isovalues: List of N isovalue thresholds.
        num_bins: Number of bins for the distance histogram.
        dist_range: Tuple (min_dist, max_dist) for histogram binning.
        
    Returns:
        smv: N x N numpy array of similarity values.
    """
    N = len(isovalues)
    smv = np.zeros((N, N))
    
    # Precompute distance fields for all isovalues
    distance_fields = []
    for h in isovalues:
        # Compute signed distance field for isosurface h
        # This is a placeholder for the specific SDF algorithm (e.g., fast marching)
        # [G] Implementation of SDF computation is not detailed in [S1], 
        # but must produce a field D(x) for all voxels.
        sdf = compute_signed_distance_field(volume, h) 
        distance_fields.append(sdf)
        
    # Define histogram bins
    bins = np.linspace(dist_range[0], dist_range[1], num_bins + 1)
    
    total_voxels = volume.size
    
    for i in range(N):
        for j in range(i, N): # Symmetry allows optimization
            d_p = distance_fields[i]
            d_q = distance_fields[j]
            
            # Flatten distance fields
            d_p_flat = d_p.flatten()
            d_q_flat = d_q.flatten()
            
            # Compute joint histogram
            # We only count voxels where both distances are within range to avoid edge artifacts
            mask = (d_p_flat >= dist_range[0]) & (d_p_flat <= dist_range[1]) & \
                   (d_q_flat >= dist_range[0]) & (d_q_flat <= dist_range[1])
            
            valid_p = d_p_flat[mask]
            valid_q = d_q_flat[mask]
            valid_count = len(valid_p)
            
            if valid_count == 0:
                smv[i, j] = 0.0
                smv[j, i] = 0.0
                continue
                
            # Histogram 2D
            hist, _, _ = np.histogram2d(valid_p, valid_q, bins=bins)
            
            # Normalize to get joint probability
            p_xy = hist / valid_count
            
            # Marginal probabilities
            p_x = np.sum(p_xy, axis=1)
            p_y = np.sum(p_xy, axis=0)
            
            # Compute Entropies
            # Helper to compute entropy, handling 0 log 0
            def entropy(p):
                p = p[p > 0]
                return -np.sum(p * np.log2(p))
            
            h_x = entropy(p_x)
            h_y = entropy(p_y)
            h_xy = entropy(p_xy.flatten())
            
            # Compute Normalized Mutual Information
            # Formula from [S1] Eq 7 (inferred standard symmetric NMI)
            if (h_x + h_y) == 0:
                similarity = 0.0
            else:
                mi = h_x + h_y - h_xy
                similarity = 2.0 * mi / (h_x + h_y)
                
            smv[i, j] = similarity
            smv[j, i] = similarity
            
    return smv

def compute_signed_distance_field(volume, isovalue):
    """
    [G] Placeholder for SDF computation.
    In practice, this uses a library or algorithm to compute the distance from each voxel
    to the isosurface defined by 'isovalue' in 'volume'.
    """
    # This function is not defined in [S1]. 
    # A common approach is using the Fast Marching Method or Euclidean Distance Transform.
    # For the purpose of this article, we assume this function exists and returns a 3D array.
    pass
```

**Edge Cases and Pitfalls:**
*   **Zero Entropy:** If an isosurface is empty or the distance field is constant (e.g., all zeros), entropy is zero. The NMI formula must handle division by zero. [S1]
*   **Distance Range:** The choice of `dist_range` for binning affects the histogram. Voxels outside this range are typically ignored or clipped. [S1]
*   **Symmetry:** The map is symmetric. Computing only the upper triangle is sufficient. [S1]
*   **Diagonal:** The diagonal elements are always 1.0. [S1]

## Subtopics

### Isosurface Similarity Distribution (SDV)
The Isosurface Similarity Distribution (SDV) is a vector derived from the ISM that describes the average similarity for each individual isosurface. It is computed by summing the rows (or columns) of the ISM and normalizing by the number of isovalues $|V|$:
$$ SDV(i) = \frac{1}{|V|} \sum_{j=1}^{|V|} SMV(i, j) $$
Peaks in the SDV correspond to isosurfaces that are most similar to others in the dataset, while valleys indicate regions of rapid structural change. To efficiently evaluate the SDV for arbitrary continuous subranges of isovalues (rather than the full set), a summed-area table of the similarity map should be used. [S1]

## Operating characteristics
*   **Time Complexity:** $O(N^2 \cdot V)$, where $N$ is the number of isovalues and $V$ is the number of voxels. The dominant cost is computing the joint histogram for each pair of isosurfaces.
*   **Memory Complexity:** $O(N^2 + V)$. The ISM matrix requires $N^2$ storage. Distance fields require $V$ storage per isosurface (or can be computed on the fly to save memory at the cost of time).
*   **Strengths:** Provides a comprehensive overview of structural similarity across all thresholds. Robust to noise, as shown in [S1]. Visualizes relationships that histograms miss.
*   **Limitations:** Computationally expensive for large $N$ or large volumes. Requires robust signed distance field computation.
*   **Visualization:** The ISM is typically visualized as a grayscale intensity matrix where white represents a similarity of zero and black represents a similarity of one. [S1]

## Background
The Isosurface Similarity Map was introduced by S. Bruckner and T. Möller. The method is based on the concept of measuring the similarity between isosurfaces using the normalized mutual information of their distance fields. It was developed to provide a more holistic view of volume data structure than traditional scalar histograms. [S1]

## Sources
[S1] Isosurface Similarity Maps, https://doi.org/10.1111/j.1467-8659.2009.01689.x, fetched 2026-10-27
