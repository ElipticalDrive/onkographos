# PCA-based Clustering
Also called: Heckel et al. clustering, Principal Component Analysis clustering

## Overview
PCA-based clustering partitions a set of scattered points into well-shaped clusters by aligning the splitting hyperplane with the principal axes of the point distribution. It solves the problem of generating data-dependent hierarchies for surface reconstruction or volume representation where cluster shape quality is critical. It is distinguished from basic adaptive clustering by its use of eigenanalysis to determine splitting directions, ensuring splits are orthogonal to the longest axis of the point cloud, and from affine-invariant approaches by its standard (non-normalized) covariance analysis which may produce elongated clusters on curved data [S1, unofficial].

## How it works
**Inputs:** A set of $n$ points $\mathbf{p}_i = (p_{i1}, \dots, p_{ik})^T$ in $k$-dimensional space, optionally with associated scalar function values $f_i$ and weights $w_i$.
**Outputs:** A splitting hyperplane defined by a normal vector and a point on the plane (typically the centroid), and the partition of points into two subsets based on the hyperplane.

**Data Structures:**
*   **Point Cloud:** List of $n$ vectors of size $k$.
*   **Covariance Matrix:** $k \times k$ symmetric matrix $A$.
*   **Eigenvectors:** $k$ orthonormal vectors $\mathbf{e}_1, \dots, \mathbf{e}_k$.
*   **Eigenvalues:** $k$ real, non-negative scalars $\lambda_1, \dots, \lambda_k$.

**Steps:**

1.  **Compute Centroid:** Calculate the mean position of the points. If weights $w_i$ are provided, use the weighted centroid.
    $$ \mathbf{c} = \frac{1}{n} \sum_{i=1}^n \mathbf{p}_i \quad \text{(unweighted)} $$
    $$ \mathbf{c} = \frac{\sum_{i=1}^n w_i \mathbf{p}_i}{\sum_{i=1}^n w_i} \quad \text{(weighted)} $$

2.  **Compute Covariance Matrix:** Calculate the scatter matrix centered at $\mathbf{c}$.
    $$ a_{jk} = \frac{1}{n} \sum_{i=1}^n (p_{ij} - c_j)(p_{ik} - c_k) \quad \text{(unweighted)} $$
    $$ a_{jk} = \frac{\sum_{i=1}^n w_i(p_{ij} - c_j)(p_{ik} - c_k)}{\sum_{i=1}^n w_i} \quad \text{(weighted)} $$

3.  **Eigenanalysis:** Compute the eigenvalues $\lambda$ and corresponding orthonormal eigenvectors $\mathbf{e}$ of the symmetric matrix $A$. Sort them such that $\lambda_{\min}$ corresponds to the smallest eigenvalue and $\lambda_{\max}$ to the largest.

4.  **Define Approximating Function:** The local distance to the reconstructed manifold is approximated by the signed distance along the direction of the smallest eigenvalue (assumed to be the surface normal):
    $$ \tilde{f}_k^j(\mathbf{x}) = (\mathbf{x} - \mathbf{c}) \cdot \mathbf{e}_{\lambda_{\min}} $$

5.  **Define Splitting Hyperplane:** The splitting plane passes through the centroid $\mathbf{c}$ and is orthogonal to the eigenvector with the largest eigenvalue $\mathbf{e}_{\lambda_{\max}}$. This ensures the split is orthogonal to the longest axis of the cluster.
    $$ \text{Normal } \mathbf{n} = \mathbf{e}_{\lambda_{\max}} $$
    $$ \text{Plane Equation: } (\mathbf{x} - \mathbf{c}) \cdot \mathbf{e}_{\lambda_{\max}} = 0 $$

6.  **Partition:** Assign each point $\mathbf{p}_i$ to a subset based on the sign of $(\mathbf{p}_i - \mathbf{c}) \cdot \mathbf{e}_{\lambda_{\max}}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pca_cluster(points, weights=None):
    """
    Performs PCA-based clustering on a set of points.
    Returns the splitting normal and centroid.
    """
    n = len(points)
    k = len(points[0])
    
    # Step 1: Compute Centroid
    if weights is None:
        weights = [1.0] * n
    sum_weights = sum(weights)
    centroid = [0.0] * k
    for i in range(n):
        for j in range(k):
            centroid[j] += weights[i] * points[i][j]
    for j in range(k):
        centroid[j] /= sum_weights
        
    # Step 2: Compute Covariance Matrix
    cov = [[0.0] * k for _ in range(k)]
    for i in range(n):
        diff = [points[i][j] - centroid[j] for j in range(k)]
        for j in range(k):
            for l in range(k):
                cov[j][l] += weights[i] * diff[j] * diff[l]
    for j in range(k):
        for l in range(k):
            cov[j][l] /= sum_weights
            
    # Step 3: Eigenanalysis
    # Compute eigenvalues and eigenvectors of symmetric matrix 'cov'
    eigenvalues, eigenvectors = compute_eigen_decomposition(cov)
    
    # Find indices of min and max eigenvalues
    min_idx = argmin(eigenvalues)
    max_idx = argmax(eigenvalues)
    
    # Step 4 & 5: Define Splitting Hyperplane
    # Normal is eigenvector corresponding to largest eigenvalue
    splitting_normal = eigenvectors[:, max_idx]
    
    return centroid, splitting_normal
```

**Parameters:**
*   **Weights ($w_i$):** Not given in the sources as a default. If function values $f_i$ are to be considered, $w_i$ is defined by the approximation error [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Curved Data:** The basic process ignores associated function values. If the function is curved in only one direction, this method generates long and skinny clusters [S1, unofficial].
*   **Recursive Splitting:** The affine-invariant variant (scaling by $\sqrt{1/\lambda_i}$) may work well on densely sampled, smooth data but provides poor approximation behavior in regions where clusters are recursively split along similar directions [S1, unofficial].
*   **Degenerate Input:** If points are collinear or coplanar, eigenvalues may be zero, leading to undefined normals for the splitting plane.

## Subtopics

### Affine-Invariant PCA Clustering
A variant that transforms points into a PCA-based coordinate system and scales coordinates by $\sqrt{1/\lambda_i}$ to achieve invariance under shear and other affine transformations. It is chosen when geometric invariance to non-uniform scaling is required, but may fail on data with recursive splitting along similar directions [S1, unofficial]. [Affine-Invariant Clustering](link-to-affine-invariant.md)

## Operating characteristics
*   **Time Complexity:** Dominated by the eigenanalysis of the $k \times k$ covariance matrix. For fixed dimension $k$, this is $O(n)$ per cluster for computing the matrix, plus constant time for eigen-decomposition.
*   **Memory Complexity:** $O(nk)$ to store the point cloud and $O(k^2)$ for the covariance matrix.
*   **Strengths:** Produces well-shaped clusters by splitting orthogonal to the longest axis. Invariant under translation, rotation, and uniform scaling [S1, unofficial].
*   **Limitations:** Not invariant under general affine transformations (unless the affine-invariant variant is used). Ignores function values by default, leading to poor results on curved data if weights are not applied. The affine-invariant variant can produce poor approximations in specific recursive splitting scenarios [S1, unofficial].
*   **When to use:** Densely sampled, smooth data sets where cluster shape quality is prioritized over function-value-aware splitting.
*   **When not to use:** Data with significant curvature where function-value-aware splitting (e.g., bisector methods) yields fewer clusters or better approximation [S1, unofficial].

## Background
The PCA-based clustering approach for surface reconstruction was introduced by Heckel et al. [S1, unofficial]. It matches the general definition of adaptive clustering by using an approximating function to define local distance and a splitting plane for refinement. The method leverages the property that for a $s$-manifold, the eigenvector with the smallest eigenvalue of the covariance matrix is locally orthogonal to the manifold, serving as a normal estimate [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
