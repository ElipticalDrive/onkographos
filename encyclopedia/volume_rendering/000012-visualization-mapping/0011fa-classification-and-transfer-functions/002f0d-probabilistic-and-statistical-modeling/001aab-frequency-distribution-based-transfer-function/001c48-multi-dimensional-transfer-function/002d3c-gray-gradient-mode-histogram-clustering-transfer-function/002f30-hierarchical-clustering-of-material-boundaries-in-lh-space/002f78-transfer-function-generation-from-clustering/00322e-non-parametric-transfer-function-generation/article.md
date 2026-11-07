# Non-Parametric Transfer Function Generation

Also called: Variable Kernel Density Estimation for TF Clustering

## Overview
Non-Parametric Transfer Function Generation is a clustering method for 2D histograms (conventional or abstracted) that groups data based on neighborhood information using variable kernel density estimation [S1, unofficial]. It differs from discrete histogram-based clustering by operating on continuous density estimates, thereby avoiding spurious peaks and preserving spatial relationships of neighboring data points [S1, unofficial]. Unlike frequency binning methods that consider only density magnitude, this method incorporates position in feature space via adaptive kernel widths, preventing disjoint segments for the same density value [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ data points $\{X_1, X_2, ..., X_N\}$ representing voxels in a 2D feature space (e.g., Gray-Gradient Mode Histogram).
*   **Output:** A probability density estimate $\hat{f}_h(x)$ over the feature space, which identifies clusters as local maxima in the density distribution.

**Data Structures**
*   **Samples:** An array of $N$ points, where each point $X_i$ is a vector in the 2D feature space.
*   **Parameters:**
    *   $N$: Total number of samples (voxels).
    *   $k$: Index of the nearest neighbor used to scale the kernel radius.
    *   $d_{i,k}$: Multi-dimensional smoothing parameter for point $X_i$.

**Steps**
1.  **Calculate Smoothing Parameters:** For each sample $X_i$, compute $d_{i,k}$, the distance from $X_i$ to its $k$-th nearest neighbor in the set of $N-1$ other data points.
2.  **Select Kernel:** Use the Epanechnikov kernel $K(u)$ for computational efficiency.
3.  **Compute Density Estimate:** Calculate the probability density $\hat{f}_h(x)$ at any location $x$ in the feature space using the variable kernel formula.
4.  **Identify Clusters:** Locate local maxima in the resulting density surface $\hat{f}_h(x)$. Each local maximum represents a potential volumetric feature cluster.

**Formulas**
The probability density estimate $\hat{f}_h(x)$ at location $x$ is:
$$ \hat{f}_h(x) = \frac{1}{N} \sum_{i=1}^{N} \frac{1}{d_{i,k}} K\left(\frac{x - X_i}{d_{i,k}}\right) $$

Where:
*   $N$ is the total number of samples in the histogram feature space [S1, unofficial].
*   $d_{i,k}$ is the multi-dimensional smoothing parameter, defined as the distance from the $i$-th sample $X_i$ to its $k$-th nearest neighbor in the set of $N-1$ data points [S1, unofficial].
*   $K(u)$ is the kernel function. The Epanechnikov kernel is used to reduce calculation time [S1, unofficial]:
    $$ K(u) = \frac{3}{4}(1 - u^2) \cdot \mathbb{I}(\|u\| \le 1) $$
    where $\mathbb{I}(\cdot)$ is the indicator function, evaluating to 1 if the inequality is true and 0 otherwise [S1, unofficial].

**Parameters**
*   $k$: The index of the nearest neighbor. Set to $k = \lfloor \sqrt{N} \rfloor$ as a rule of thumb approximation for optimal density estimation fitting [S1, unofficial].

**Pitfalls**
*   **Sparse Regions:** Data points in sparse regions require flatter kernels to maintain density estimation accuracy. This is handled by scaling the kernel width proportional to $d_{i,k}$ (distance to $k$-th nearest neighbor) [S1, unofficial].
*   **Boundary Effects:** The method assumes the feature space is bounded; points near the edge may have underestimated density due to truncated kernel support [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def epanechnikov_kernel(u_norm_sq):
    """
    Epanechnikov kernel K(u) = 3/4 * (1 - u^2) if ||u|| <= 1, else 0.
    Input: squared norm of u (||u||^2).
    """
    if u_norm_sq <= 1.0:
        return 0.75 * (1.0 - u_norm_sq)
    return 0.0

def compute_density_estimate(samples, query_point, k):
    """
    Compute the non-parametric density estimate at query_point.
    
    Args:
        samples: List of N 2D vectors (X_i).
        query_point: 2D vector (x) where density is estimated.
        k: Index of nearest neighbor for bandwidth scaling.
        
    Returns:
        Float: The density estimate f_hat(x).
    """
    N = len(samples)
    density = 0.0
    
    # Precompute distances to find k-th nearest neighbor for each sample
    # Note: In a full implementation, this would be done efficiently (e.g., KD-tree).
    # Here we compute d_i,k for each i.
    d_i_k = []
    for i, X_i in enumerate(samples):
        # Calculate distances from X_i to all other samples
        dists = []
        for j, X_j in enumerate(samples):
            if i == j:
                continue
            diff = np.array(query_point) - np.array(X_i) # Wait, d_i,k is distance between samples, not to query
            # Correction: d_i,k is distance from X_i to its k-th nearest neighbor in the dataset
            pass 
        
    # Re-structure for clarity: First pass to find d_i,k for all i
    d_i_k_values = []
    for i, X_i in enumerate(samples):
        dists_to_others = []
        for j, X_j in enumerate(samples):
            if i == j:
                continue
            dist = np.linalg.norm(np.array(X_i) - np.array(X_j))
            dists_to_others.append(dist)
        dists_to_others.sort()
        # k-th nearest neighbor (1-indexed in formula context, but list is 0-indexed)
        # The formula says "k-th nearest neighbor". If k=1, it's the closest.
        # Python list index k-1
        if k <= len(dists_to_others):
            d_i_k_values.append(dists_to_others[k-1])
        else:
            d_i_k_values.append(dists_to_others[-1]) # Fallback if k > N-1

    # Second pass: Compute density sum
    for i, X_i in enumerate(samples):
        d_i_k = d_i_k_values[i]
        if d_i_k == 0:
            # Avoid division by zero; handle degenerate case
            # If points are identical, standard practice is to skip or use epsilon
            continue
            
        # Calculate scaled vector u = (x - X_i) / d_i,k
        diff = np.array(query_point) - np.array(X_i)
        u = diff / d_i_k
        u_norm_sq = np.dot(u, u)
        
        # Kernel value
        K_u = epanechnikov_kernel(u_norm_sq)
        
        # Add to sum: (1/N) * (1/d_i,k) * K(u)
        density += (1.0 / d_i_k) * K_u
        
    density /= N
    return density

def generate_clusters(samples, k, resolution=100):
    """
    Generate a density grid and find local maxima.
    """
    N = len(samples)
    # Determine bounds
    min_vals = np.min(samples, axis=0)
    max_vals = np.max(samples, axis=0)
    
    # Create grid
    x_grid = np.linspace(min_vals[0], max_vals[0], resolution)
    y_grid = np.linspace(min_vals[1], max_vals[1], resolution)
    
    density_grid = np.zeros((resolution, resolution))
    
    for i, x in enumerate(x_grid):
        for j, y in enumerate(y_grid):
            query_point = [x, y]
            density_grid[i, j] = compute_density_estimate(samples, query_point, k)
            
    # Find local maxima (simplified)
    # In practice, use scipy.ndimage.maximum_filter or similar
    # This is a placeholder for the clustering step
    return density_grid
```

## Subtopics

### [Topology-based Non-Parametric Clustering](7417-topology-based-non-parametric-clustering.md)
Topology-based Non-Parametric Clustering decomposes a 2D density plot feature space into non-overlapping "valley cells" separated by "valley lines," where each cell contains a local maximum representing a potential volumetric feature. It differs from discrete histogram-based clustering by operating on continuous density plots, thereby avoiding spurious peaks/valleys and preserving the spatial relationships of neighboring data points. Unlike frequency binning methods that consider only density magnitude, this method incorporates position in feature space via topological information, preventing disjoint segments for the same density value.

## Operating characteristics
*   **Time Complexity:** $O(N^2)$ for computing all pairwise distances to find $k$-th nearest neighbors if done naively. Density evaluation at a single point is $O(N)$. For a grid of size $M$, total time is $O(M \cdot N)$. Using spatial indexing (e.g., KD-tree) can reduce neighbor search to $O(N \log N)$ [G].
*   **Memory Complexity:** $O(N)$ to store samples and distances. $O(M)$ for the density grid if precomputed.
*   **Strengths:** Adapts to local data density, handling sparse and dense regions differently. Avoids artifacts from fixed-bin histograms.
*   **Limitations:** Computationally expensive for large $N$ without optimization. Sensitive to the choice of $k$.
*   **When to use:** When the feature space has varying density and standard histogram binning creates misleading clusters.
*   **When not to use:** For very large datasets where real-time performance is critical and approximate methods are acceptable.

## Background
Originated from the work of Maciejewski et al. [12] (cited in [S1, unofficial]), specifically in the context of abstract feature space representation for volumetric transfer function exploration. The method was introduced to provide users with a means to group data in the 2D histogram by areas of similar value. It builds on variable kernel density estimation techniques [17, 19] cited in [S1, unofficial].

## Sources
[S1] Abstract Feature Space Representation for Volumetric Transfer Function Exploration, https://doi.org/10.4230/dfu.vol2.sciviz.2011.212, fetched 2026-10-26 (unofficial)
