# PCA-based Clustering

Also called: Heckel et al. clustering, Principal Component Analysis clustering

## Overview
PCA-based Clustering is a method for generating adaptive clusters from scattered point data by partitioning a set of samples recursively based on the geometric distribution of their spatial coordinates [S1, unofficial]. It solves the problem of creating data-dependent hierarchies that respect the intrinsic topological and geometric structures of the input cloud, ensuring well-shaped clusters by splitting orthogonal to the direction of maximum variance [S1, unofficial]. It sets itself apart from its siblings by using Principal Component Analysis to define a local coordinate system and splitting plane, whereas [Adaptive Clustering with Error-Bisector Splitting](6211-adaptive-clustering-with-error-bisector-splitting.md) uses error metrics and [Hierarchical Density-based Clustering](10952-hierarchical-density-based-clustering.md) relies on density estimation [S1, unofficial].

## How it works
**Inputs:** A set of $n$ points $\mathbf{p}_i = (p_{i1}, \dots, p_{ik})^T$ in $k$-dimensional space, and optionally associated scalar function values $f_i$ with weights $w_i$ [S1, unofficial].
**Outputs:** A hierarchical tree of clusters, where each node represents a subset of points [G].

**Data Structures:**
*   **Point Cloud:** List of $n$ vectors of size $k$ [G].
*   **Cluster Node:** Contains a list of indices to points, a centroid $\mathbf{c}$, a covariance matrix $A$, and eigenvectors/eigenvalues [G].

**Steps:**
1.  **Initialization:** Assign all input points to the root cluster [G].
2.  **Centroid Calculation:** Compute the centroid $\mathbf{c}$ of the points in the current cluster. If weights $w_i$ are provided, use the weighted centroid [S1, unofficial]:
    $$ \mathbf{c} = \frac{\sum_{i=1}^n w_i \mathbf{p}_i}{\sum_{i=1}^n w_i} $$
    If no weights are used, the standard centroid is [S1, unofficial]:
    $$ \mathbf{c} = \frac{1}{n} \sum_{i=1}^n \mathbf{p}_i $$
3.  **Covariance Matrix Construction:** Compute the $k \times k$ symmetric covariance matrix $A$ with elements $a_{jk}$ [S1, unofficial]. For weighted points:
    $$ a_{jk} = \frac{\sum_{i=1}^n w_i (p_{ij} - c_j)(p_{ik} - c_k)}{\sum_{i=1}^n w_i} $$
    For unweighted points:
    $$ a_{jk} = \frac{1}{n} \sum_{i=1}^n (p_{ij} - c_j)(p_{ik} - c_k) $$
4.  **Eigenanalysis:** Perform eigenvalue decomposition on $A$ to obtain orthonormal eigenvectors $\mathbf{e}_1, \dots, \mathbf{e}_k$ and corresponding real, non-negative eigenvalues $\lambda_1, \dots, \lambda_k$ [S1, unofficial].
5.  **Splitting Plane Determination:** Identify the eigenvector $\mathbf{e}_{\lambda_{max}}$ associated with the largest eigenvalue $\lambda_{max}$. The splitting hyperplane is defined to pass through the centroid $\mathbf{c}$ and be orthogonal to $\mathbf{e}_{\lambda_{max}}$ [S1, unofficial]. This ensures the cluster is split orthogonal to its longest axis [S1, unofficial].
6.  **Partitioning:** Assign each point $\mathbf{p}_i$ in the cluster to one of the two sub-clusters based on the sign of the projection onto the splitting normal $\mathbf{e}_{\lambda_{max}}$. Specifically, if $(\mathbf{p}_i - \mathbf{c}) \cdot \mathbf{e}_{\lambda_{max}} > 0$, assign to one child; otherwise, to the other [S1, unofficial].
7.  **Recursion:** Repeat steps 2–6 for each sub-cluster until a stopping criterion is met [G].

**Approximating Function:**
The method defines an approximating function $\tilde{f}_k^j(\mathbf{x})$ representing the local signed distance to the reconstructed manifold, using the eigenvector $\mathbf{e}_{\lambda_{min}}$ with the smallest eigenvalue [S1, unofficial]:
$$ \tilde{f}_k^j(\mathbf{x}) = (\mathbf{x} - \mathbf{c}) \cdot \mathbf{e}_{\lambda_{min}} $$
This vector is assumed to be locally orthogonal to the manifold if the points define a surface [S1, unofficial].

**Parameters:**
*   **Weights ($w_i$):** Optional. If associated function values $f_i$ are ignored, $w_i=1$ for all $i$ [S1, unofficial]. To incorporate function values, $w_i$ should be defined by the approximation error [S1, unofficial].
*   **Stopping Criterion:** Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Shear Invariance:** The basic clustering process is not invariant under shear transformations [S1, unofficial].
*   **Ignoring Function Values:** The standard process ignores associated function values $f_i$ [S1, unofficial].
*   **Degenerate Input:** If points are collinear or coplanar, the covariance matrix may be singular or have zero eigenvalues [G]. The source notes eigenvalues are real and positive for the unweighted case, implying non-degenerate configurations or implicit regularization [S1, unofficial].

## Subtopics

### [Affine-Invariant Clustering](5762-affine-invariant-clustering.md)
Affine-Invariant Clustering is a method for generating adaptive clusters from scattered point data that remains invariant under general affine transformations, including translation, rotation, uniform scaling, and shear. It solves the limitation of standard PCA-based clustering, which is not invariant under shear transformations. By transforming data into a PCA-based coordinate system and normalizing axes by eigenvalues, it produces well-shaped clusters that are robust to affine distortions, distinguishing it from its parent method, [PCA-based Clustering](5048-pca-based-clustering.md), which lacks shear invariance.

### Iterative Cluster Discriminant Optimization
This variant refines cluster assignments by optimizing a discriminant ratio $T_J$ [S2, unofficial]. In an iterative round, each pixel (data point) is temporarily removed from its home cluster and assigned to each of the other clusters one by one [S2, unofficial]. For each temporary assignment, the differences in means and variances are renewed and the discriminant ratio is recalculated [S2, unofficial]. A pixel is permanently detached and reassigned to another cluster only if such reassignment causes an increase in the ratio, promoting larger inter-cluster distance and smaller intra-cluster distance [S2, unofficial]. Statistical optimization ends when ratio alteration becomes ignorable [S2, unofficial]. The discriminant ratio $T_J$ is calculated as [S2, unofficial]:
$$ T_J = \frac{\sum_{k_i=1} \sum_{k_j>i} (m_{ip} - m_{jp})(m_{ip} - m_{jp})}{\sum_{k_i=1} v_{ip}} $$
Note: The denominator in the source text appears garbled as `(∑𝑘𝑖=1 𝑣𝑖𝑝 𝑣𝑖𝑝 )`, likely representing the sum of variances or a normalization factor involving $v_{ip}$ [S2, unofficial].

### Variance-based Intra-leaf Clustering
This variant adaptively fits anisotropic 3D Gaussians to sparse volumetric data by recursively splitting voxel regions based on variance until local homogeneity is achieved [S3, unofficial]. The algorithm first computes the centroid and average value of the active voxels in a leaf node [S3, unofficial]. It then computes the population variance, which defines how much the voxel values differ from the average [S3, unofficial]:
$$ \text{variance} = \frac{1}{N} \sum_{i=1}^N \| \mathbf{x}_i - \boldsymbol{\mu} \|^2 $$
where $N$ is the number of active voxels, $\mathbf{x}_i$ is the world-space position of the $i$-th voxel, and $\boldsymbol{\mu}$ is the centroid [S3, unofficial]. If the variance is below a threshold, the method emits a Gaussian for the block of voxels [S3, unofficial]. If the variance is large, the block is split into two along the axis with the most variance and each half is processed recursively [S3, unofficial].

## Operating characteristics
**Time Complexity:** Not given in the sources [G].
**Memory Complexity:** Not given in the sources [G].
**Measured Performance:** Not given in the sources [G].
**Strengths:** Produces well-shaped clusters by splitting orthogonal to the longest axis; provides a local coordinate system aligned with the data manifold; simple to implement using standard linear algebra routines [S1, unofficial].
**Limitations:** Not invariant under shear transformations; ignores associated function values unless weights are explicitly defined [S1, unofficial].
**When to use:** Not given in the sources [G].
**When not to use:** Not given in the sources [G].

## Background
PCA-based clustering was introduced by Heckel et al. for surface reconstruction and adaptive resolution data management [S1, unofficial]. It matches the general definition of adaptive clustering by using PCA to construct an approximating function and estimate a splitting plane [S1, unofficial]. The approach uses the eigenvector with the smallest eigenvalue as a local estimate for the surface normal and the eigenvector with the largest eigenvalue to define the splitting plane [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
[S2] Segmentation Assisted Object Distinction for Direct Volume Rendering, https://doi.org/10.22044/jadm.2019.7207.1854, fetched 2026-10-29 (unofficial)
[S3] 3D Gaussian Particle Approximation of VDB Datasets: A Study for Scientific Visualization, https://www.emergentmind.com/papers/2504.04857, fetched 2026-10-29 (unofficial)
