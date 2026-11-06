# Adaptive Clustering

Also called: Data-dependent hierarchical clustering, Hierarchical Adaptive Clustering

## Overview
Adaptive clustering is a technique for generating a data-dependent hierarchy that represents the inherent topological and geometric structures of scattered data by recursively partitioning a set of samples into smaller groups (clusters) based on spatial adjacency or correlation of associated function values [S1, unofficial]. Unlike mesh-reduction algorithms, it does not require a grid structure connecting the given data points, making it suitable for unstructured datasets [S1, unofficial]. It builds on [Hierarchical Voronoi Diagrams](4112-hierarchical-voronoi-diagrams.md) by using the hierarchy to define approximation regions and error metrics for multiresolution representation [S1, unofficial].

## How it works
**Inputs:** A set $P$ of $n$ points in $s$-dimensional space with associated $t$-dimensional function values, defined as $P = \{(\mathbf{p}_i, f_i) \mid \mathbf{p}_i \in \mathbb{R}^s, f_i \in \mathbb{R}^t, i = 1, \dots, n\}$ [S1, unofficial]. The domain $D \subset \mathbb{R}^s$ is assumed to be compact and contain all points $\mathbf{p}_i$ [S1, unofficial].

**Outputs:** A hierarchy of levels $L_j$ for $j = 0, 1, \dots$, where each level is defined as:
$$L_j = \{(\tau_k^j, \tilde{f}_k^j, \varepsilon_k^j) \mid k = 1, \dots, n_j\}$$
Here, $\tau_k^j$ are tiles forming a partition of domain $D$, $\tilde{f}_k^j$ are approximating functions for the tile, and $\varepsilon_k^j$ estimate the approximation error [S1, unofficial].

**Data Structures:**
*   **Point Set:** List of tuples $(\mathbf{p}_i, f_i)$ [S1, unofficial].
*   **Level Structure:** List of cluster objects, each containing:
    *   `tile`: The spatial region (partition of $D$) [G].
    *   `approx`: The function $\tilde{f}_k^j$ mapping points in the tile to $\mathbb{R}^t$ [G].
    *   `error`: The scalar residual $\varepsilon_k^j$ [S1, unofficial].
    *   `points`: List of indices of points contained in the tile [G].

**Parameters:**
*   $p$: Exponent for the error norm, $p \in [1, \infty]$ [S1, unofficial]. For $p=\infty$, the residual is the maximal error within the corresponding tile [S1, unofficial].
*   $n_{max}$: Maximum number of points determining an approximating function $\tilde{f}_k^j$ [S1, unofficial]. If a tile contains more than $n_{max}$ points, $n_{max}$ points are selected randomly [S1, unofficial].
*   Stopping criteria: Prescribed number of clusters or a global error bound $\varepsilon_{target}$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Start with a coarse approximation level $L_0$ [S1, unofficial].
2.  **Error Calculation:** For each cluster $k$ in the current level $j$, calculate the local residual $\varepsilon_k^j$ using the $L_p$ norm:
    $$\varepsilon_k^j = \left( \frac{1}{n_k^j} \sum_{\mathbf{p}_i \in \tau_k^j} \|\tilde{f}_k^j(\mathbf{p}_i) - f_i\|^p \right)^{\frac{1}{p}}$$
    where $n_k^j$ is the number of points in tile $\tau_k^j$ and $\|\cdot\|$ is the Euclidean norm in $\mathbb{R}^t$ [S1, unofficial].
3.  **Global Error Check:** Calculate the global error for level $j$:
    $$\varepsilon^j = \left( \frac{1}{n} \sum_{k=1}^{n_j} n_k^j (\varepsilon_k^j)^p \right)^{\frac{1}{p}}$$
    If $\varepsilon^j$ satisfies the prescribed error bound or the number of clusters meets the limit, terminate [S1, unofficial].
4.  **Refinement:** Refine all tiles $\tau_k^j$ corresponding to residuals $\varepsilon_k^j$ greater than a specified threshold, rather than refining one by one, allowing exponential growth of clusters with level index $j$ [S1, unofficial].
5.  **Iteration:** Repeat steps 2–4 until convergence. Convergence is reached when every tile contains exactly one data point or the number of points in every tile is sufficiently low providing zero residuals [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_clustering(points, function_values, p=2, max_clusters=None, error_threshold=None, residual_threshold=None, n_max=None):
    """
    points: list of s-dimensional coordinate tuples/lists
    function_values: list of t-dimensional value tuples/lists
    p: norm exponent, p in [1, inf]
    max_clusters: int or None
    error_threshold: float or None
    residual_threshold: float or None, tiles with epsilon_k > this are refined
    n_max: int or None, max points used to compute approximation per tile
    """
    n = len(points)
    t_dim = len(function_values[0]) if n > 0 else 0
    
    # Level 0: one cluster containing all points [S1, unofficial]
    current_level = [{'indices': list(range(n)), 'error': 0.0}]
    
    iteration = 0
    while True:
        global_error_sum = 0.0
        clusters_to_refine = []
        
        for cluster in current_level:
            indices = cluster['indices']
            n_k = len(indices)
            if n_k == 0:
                cluster['error'] = 0.0
                continue
            
            # Select up to n_max points for approximation [S1, unofficial]
            sample_indices = indices
            if n_max is not None and n_k > n_max:
                # Sources specify random selection; deterministic fallback for reproducibility
                sample_indices = indices[:n_max]
            
            # Compute approximation f_tilde (mean of function values) [G]
            f_tilde = [0.0] * t_dim
            for idx in sample_indices:
                fv = function_values[idx]
                for d in range(t_dim):
                    f_tilde[d] += fv[d]
            for d in range(t_dim):
                f_tilde[d] /= len(sample_indices)
            
            # Calculate local residual epsilon_k
            sum_err_p = 0.0
            for idx in indices:
                fv = function_values[idx]
                diff_sq = 0.0
                for d in range(t_dim):
                    diff_sq += (f_tilde[d] - fv[d])**2
                sum_err_p += diff_sq**(p/2)
            epsilon_k = (sum_err_p / n_k)**(1/p)
            cluster['error'] = epsilon_k
            
            # Accumulate for global error formula
            global_error_sum += n_k * (epsilon_k ** p)
            
            # Refinement condition: refine if n_k > 1 and error exceeds threshold
            if n_k > 1:
                if residual_threshold is None or epsilon_k > residual_threshold:
                    if max_clusters is None or len(current_level) < max_clusters:
                        clusters_to_refine.append(cluster)
        
        global_error = (global_error_sum / n)**(1/p)
        
        # Check stopping criteria [S1, unofficial]
        if error_threshold is not None and global_error <= error_threshold:
            break
        if max_clusters is not None and len(current_level) >= max_clusters:
            break
        if not clusters_to_refine:
            break
            
        # Refine clusters [S1, unofficial]
        next_level = []
        for cluster in current_level:
            if cluster in clusters_to_refine:
                sub_clusters = split_cluster(cluster, points)
                next_level.extend(sub_clusters)
            else:
                next_level.append(cluster)
        current_level = next_level
        iteration += 1
        
        # Safety break
        if iteration > 1000:
            break
            
    return current_level

def split_cluster(cluster, points):
    """
    Generic split: divide points into two halves based on the first dimension median.
    [G] Specific variants (e.g., PCA or error-bisector) override this with geometric splitting.
    """
    indices = cluster['indices']
    if len(indices) <= 1:
        return [{'indices': [i], 'error': 0.0} for i in indices]
    
    sorted_indices = sorted(indices, key=lambda i: points[i][0])
    mid = len(sorted_indices) // 2
    return [
        {'indices': sorted_indices[:mid], 'error': 0.0},
        {'indices': sorted_indices[mid:], 'error': 0.0}
    ]
```

## Subtopics

### [PCA-based Clustering](5048-pca-based-clustering.md)
PCA-based clustering partitions a set of scattered points into well-shaped clusters by aligning the splitting hyperplane with the principal axes of the point distribution. It solves the problem of generating data-dependent hierarchies for surface reconstruction or volume representation where cluster shape quality is critical. It is distinguished from basic adaptive clustering by its use of eigenanalysis to determine splitting directions, ensuring splits are orthogonal to the longest axis of the point cloud, and from affine-invariant approaches by its standard covariance analysis which may produce elongated clusters on curved data. The centroid $\mathbf{c}$ is calculated as $\mathbf{c} = \frac{1}{n} \sum_{i=1}^n \mathbf{p}_i$ [G]. The covariance matrix $a_{jk} = \frac{1}{n} \sum_{i=1}^n (p_{ij} - c_j)(p_{ik} - c_k)$ is eigen-decomposed [G].

### [Adaptive Clustering with Error-Bisector Splitting](6211-adaptive-clustering-with-error-bisector-splitting.md)
Adaptive clustering with error-bisector splitting is a method for generating a data-dependent hierarchy of clusters from scattered 2D points with associated scalar values, using quadratic polynomial approximations to guide subdivision. It solves the problem of efficiently approximating smooth surfaces with a minimal number of clusters for a given error tolerance, outperforming PCA-based splitting by producing long, skinny clusters when curvature is directional. Unlike PCA-based clustering, which ignores function values, this method weights spatial distribution by approximation residuals.

## Operating characteristics
*   **Time Complexity:** Top-down construction allows coarse levels to be computed faster than fine levels [S1, unofficial]. The base computational cost is $O(n \log m)$ for $n$ data points and $m$ clusters [S1, unofficial]. Modifications such as threshold-based refinement and point bounding reduce the cost to expected linear time for large-scale datasets [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of clusters and points, as the hierarchical structure stores approximations and residuals for each tile [G].
*   **Strengths:** Data-dependent adaptation to geometric complexity; no grid structure required; efficient for smooth manifolds [S1, unofficial].
*   **Limitations:** Requires careful choice of error norm; performance depends on data distribution; PCA variants assume local manifold structure [S1, unofficial].
*   **When to use:** Scattered data with smooth underlying structures (e.g., terrain, isosurfaces); when grid-based methods are inapplicable [S1, unofficial].
*   **When not to use:** Highly irregular or noisy data without clear topological structure; when strict grid alignment is required [G].

## Background
Adaptive clustering was introduced as a method for multiresolution modeling of scientific visualization data, particularly for recovering topological structures of two-manifold surfaces from scattered data points in 3D space [S1, unofficial]. It was developed as an alternative to mesh-reduction algorithms, offering a "top-down" hierarchical construction [S1, unofficial]. The approach is closely related to wavelet transforms for manifold geometries, providing efficient compression and visualization at different resolutions [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
[S2] Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries, https://doi.org/10.2312/vissym/eurovis06/243-250, fetched 2026-10-26
