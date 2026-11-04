# Adaptive Clustering

Also called: Data-dependent hierarchical clustering, Hierarchical Adaptive Clustering

## Overview
Adaptive clustering is a technique for generating a data-dependent hierarchy that represents the inherent topological and geometric structures of scattered data by recursively partitioning a set of samples into smaller groups (clusters) based on spatial adjacency or correlation of associated function values [S1, unofficial]. Unlike mesh-reduction algorithms, it does not require a grid structure connecting the given data points, making it suitable for unstructured or scattered datasets [S1, unofficial]. It builds on [Hierarchical Voronoi Diagrams](4112-hierarchical-voronoi-diagrams.md) by using the hierarchy to define approximation regions and error metrics for multiresolution representation.

## How it works
**Inputs:** A set $P$ of $n$ points in $s$-dimensional space with associated $t$-dimensional function values, defined as $P = \{(\mathbf{p}_i, f_i) \mid \mathbf{p}_i \in \mathbb{R}^s, f_i \in \mathbb{R}^t, i = 1, \dots, n\}$ [S1, unofficial]. The domain $D \subset \mathbb{R}^s$ is assumed to be compact and contain all points $\mathbf{p}_i$ [S1, unofficial].

**Outputs:** A hierarchy of levels $L_j$ for $j = 0, 1, \dots$, where each level is defined as:
$$L_j = \{(\tau_k^j, \tilde{f}_k^j, \varepsilon_k^j) \mid k = 1, \dots, n_j\}$$
Here, $\tau_k^j$ are tiles forming a partition of domain $D$, $\tilde{f}_k^j$ are approximating functions for the tile, and $\varepsilon_k^j$ estimate the approximation error [S1, unofficial].

**Data Structures:**
*   **Point Set:** List of tuples $(\mathbf{p}_i, f_i)$.
*   **Level Structure:** List of cluster objects, each containing:
    *   `tile`: The spatial region (partition of $D$).
    *   `approx`: The function $\tilde{f}_k^j$ mapping points in the tile to $\mathbb{R}^t$.
    *   `error`: The scalar residual $\varepsilon_k^j$.
    *   `points`: List of indices of points contained in the tile.

**Parameters:**
*   $p$: Exponent for the error norm, $p \in [1, \infty]$. Typical values depend on the desired sensitivity to outliers; $p=2$ is common for least-squares, $p=\infty$ for max-norm [S1, unofficial].
*   Stopping criteria: Prescribed number of clusters or a global error bound $\varepsilon_{target}$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Start with a coarse approximation level $L_0$. This typically involves a single cluster containing all points or a very coarse partition [S1, unofficial].
2.  **Error Calculation:** For each cluster $k$ in the current level $j$, calculate the local residual $\varepsilon_k^j$ using the $L_p$ norm:
    $$\varepsilon_k^j = \left( \frac{1}{n_k^j} \sum_{\mathbf{p}_i \in \tau_k^j} \|\tilde{f}_k^j(\mathbf{p}_i) - f_i\|^p \right)^{\frac{1}{p}}$$
    where $n_k^j$ is the number of points in tile $\tau_k^j$ and $\|\cdot\|$ is the Euclidean norm in $\mathbb{R}^t$ [S1, unofficial].
3.  **Global Error Check:** Calculate the global error for level $j$:
    $$\varepsilon^j = \left( \frac{1}{n} \sum_{k=1}^{n_j} n_k^j (\varepsilon_k^j)^p \right)^{\frac{1}{p}}$$
    If $\varepsilon^j$ satisfies the prescribed error bound or the number of clusters meets the limit, terminate [S1, unofficial].
4.  **Refinement:** Identify clusters with large residuals (high error). Refine these clusters to create level $L_{j+1}$. Refinement involves splitting the tile $\tau_k^j$ and computing new approximations $\tilde{f}_k^{j+1}$ for the sub-clusters [S1, unofficial].
5.  **Iteration:** Repeat steps 2-4 until convergence. Convergence is reached when every tile contains exactly one data point or the number of points in every tile is sufficiently low to provide zero residuals [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_clustering(points, function_values, p=2, max_clusters=None, error_threshold=None):
    """
    points: list of numpy arrays (s-dim)
    function_values: list of numpy arrays (t-dim)
    p: norm exponent
    max_clusters: int or None
    error_threshold: float or None
    """
    n = len(points)
    # Initial level: one cluster with all points
    # Approximation could be the mean function value for simplicity, 
    # though specific methods (like PCA) define it differently.
    current_level_clusters = [Cluster(all_indices=range(n))]
    
    iteration = 0
    while True:
        # Calculate errors for current level
        global_error = 0.0
        clusters_to_refine = []
        
        for cluster in current_level_clusters:
            # Get points in this cluster
            cluster_points = [points[i] for i in cluster.indices]
            cluster_fvals = [function_values[i] for i in cluster.indices]
            n_k = len(cluster_points)
            
            # Compute approximation f_tilde for this cluster
            # [G] Default approximation: mean of function values. 
            # Specific variants (e.g., PCA) override this.
            if n_k > 0:
                f_tilde = sum(cluster_fvals) / n_k
            else:
                f_tilde = 0.0
            
            # Calculate local residual epsilon_k
            if n_k > 0:
                sum_err_p = sum(np.linalg.norm(f_tilde - f_i)**p for f_i in cluster_fvals)
                epsilon_k = (sum_err_p / n_k)**(1/p)
            else:
                epsilon_k = 0.0
            
            cluster.error = epsilon_k
            
            # Add to global error
            # Formula: (1/n) * sum(n_k * epsilon_k^p)
            global_error += n_k * (epsilon_k ** p)
            
            # Determine if refinement is needed
            # [G] Heuristic: refine if error is above a fraction of global error or max allowed
            # The source says "Refine only in cluster regions with large residuals"
            # We need a threshold. If error_threshold is global, we might refine top N% or all above avg.
            # For implementation completeness, we refine if error > 0 and we haven't hit max clusters
            # and we haven't reached single-point clusters.
            if n_k > 1:
                if max_clusters is None or len(current_level_clusters) < max_clusters:
                    clusters_to_refine.append(cluster)
        
        global_error = (global_error / n)**(1/p)
        
        # Check stopping criteria
        if error_threshold is not None and global_error <= error_threshold:
            break
        if max_clusters is not None and len(current_level_clusters) >= max_clusters:
            break
        if not clusters_to_refine:
            break
            
        # Refine clusters
        next_level_clusters = []
        for cluster in current_level_clusters:
            if cluster in clusters_to_refine:
                # Split cluster into sub-clusters
                # [G] Splitting strategy depends on the specific method (e.g., PCA splitting plane)
                # Here we assume a generic split into two sub-clusters based on spatial median
                # or principal component. For a generic implementation, we split indices.
                sub_clusters = split_cluster(cluster, points)
                next_level_clusters.extend(sub_clusters)
            else:
                next_level_clusters.append(cluster)
        
        current_level_clusters = next_level_clusters
        iteration += 1
        
        # Safety break to prevent infinite loops in generic implementation
        if iteration > 1000:
            break
            
    return current_level_clusters

def split_cluster(cluster, points):
    """
    Generic split: divide points into two halves based on the first dimension median.
    [G] In PCA-based clustering, split is along the eigenvector with max eigenvalue.
    """
    indices = cluster.indices
    if len(indices) <= 1:
        return [Cluster(indices=[i]) for i in indices]
    
    # Sort by first coordinate
    sorted_indices = sorted(indices, key=lambda i: points[i][0])
    mid = len(sorted_indices) // 2
    left_indices = sorted_indices[:mid]
    right_indices = sorted_indices[mid:]
    
    return [Cluster(indices=left_indices), Cluster(indices=right_indices)]

class Cluster:
    def __init__(self, all_indices=None, indices=None):
        if all_indices is not None:
            self.indices = all_indices
        else:
            self.indices = indices
        self.error = 0.0
```

## Subtopics

### PCA-based Clustering
A variant that uses Principal Component Analysis to define the approximating function and splitting plane. It constructs a local coordinate system for each cluster. The centroid $\mathbf{c}$ is calculated as $\mathbf{c} = \frac{1}{n} \sum_{i=1}^n \mathbf{p}_i$. The covariance matrix $a_{jk} = \frac{1}{n} \sum_{i=1}^n (p_{ij} - c_j)(p_{ik} - c_k)$ is eigen-decomposed. The approximating function is $\tilde{f}_k^j(\mathbf{x}) = (\mathbf{x} - \mathbf{c}) \cdot \mathbf{e}_{\lambda_{min}}$, where $\mathbf{e}_{\lambda_{min}}$ is the eigenvector with the smallest eigenvalue, representing the local surface normal [S1, unofficial]. The cluster is split by a hyperplane orthogonal to the eigenvector with the maximal eigenvalue $\mathbf{e}_{\lambda_{max}}$, ensuring splits are orthogonal to the longest axis [S1, unofficial]. To incorporate function values $f_i$, weighted PCA is used, where weights $w_i$ are defined by the approximation error, updating the centroid to $\mathbf{c} = \frac{\sum w_i \mathbf{p}_i}{\sum w_i}$ and covariance to $a_{jk} = \frac{\sum w_i(p_{ij} - c_j)(p_{ik} - c_k)}{\sum w_i}$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the splitting strategy. Top-down construction allows coarse levels to be computed faster than fine levels [S1, unofficial]. PCA eigen-decomposition is $O(s^3)$ per cluster, where $s$ is spatial dimension.
*   **Memory Complexity:** Proportional to the number of clusters and points. Hierarchical structure stores approximations and residuals for each tile.
*   **Strengths:** Data-dependent, adapts to geometric complexity; no grid structure required; efficient for smooth manifolds [S1, unofficial].
*   **Limitations:** Requires careful choice of error norm; PCA assumes local manifold structure; performance degrades if data is not well-clustered or noisy [S1, unofficial].
*   **When to use:** Scattered data with smooth underlying structures (e.g., terrain, isosurfaces); when grid-based methods are inapplicable [S1, unofficial].
*   **When not to use:** Highly irregular or noisy data without clear topological structure; when strict grid alignment is required.

## Background
Adaptive clustering was introduced as a method for multiresolution modeling of scientific visualization data, particularly for recovering topological structures of two-manifold surfaces from scattered data points in 3D space [S1, unofficial]. It was developed as an alternative to mesh-reduction algorithms, offering a "top-down" hierarchical construction [S1, unofficial]. The approach is closely related to wavelet transforms for manifold geometries, providing efficient compression and visualization at different resolutions [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
