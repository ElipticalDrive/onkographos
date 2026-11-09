# Affine-Invariant Clustering

Also called: Affine-invariant approach

## Overview
Affine-Invariant Clustering is a method for generating adaptive clusters from scattered point data that remains invariant under general affine transformations, including translation, rotation, uniform scaling, and shear [S1, unofficial]. It solves the limitation of standard PCA-based clustering, which is not invariant under shear transformations [S1, unofficial]. By transforming data into a PCA-based coordinate system and normalizing axes by eigenvalues, it produces well-shaped clusters that are robust to affine distortions, distinguishing it from its parent method, [PCA-based Clustering](5048-pca-based-clustering.md), which lacks shear invariance.

## How it works
**Inputs:** A set of $N$ points $P = \{p_1, \dots, p_N\}$ in $\mathbb{R}^d$.
**Outputs:** A set of clusters defined by splitting hyperplanes in the original coordinate system.

**Data Structures:**
1.  **Covariance Matrix ($\Sigma$):** A $d \times d$ symmetric matrix computed from the point set.
2.  **Eigen-decomposition:** Eigenvectors $E = [e_1, \dots, e_d]$ and eigenvalues $\Lambda = \text{diag}(\lambda_1, \dots, \lambda_d)$, where $\lambda_i \ge 0$.
3.  **Transformation Matrix ($T$):** Used to map between original and affine-invariant coordinates.

**Steps:**
1.  **Compute Covariance:** Calculate the covariance matrix $\Sigma$ of the point set $P$.
2.  **Eigen-decomposition:** Compute the eigenvectors $e_i$ and eigenvalues $\lambda_i$ of $\Sigma$.
3.  **Transform to Affine-Invariant Coordinates:**
    For each point $p_j \in P$:
    a. Center the point relative to the mean $\mu$: $p'_j = p_j - \mu$.
    b. Project onto eigenvectors: $z_{j,i} = e_i^T p'_j$.
    c. Scale by the inverse square root of the eigenvalue: $z'_{j,i} = z_{j,i} \cdot \sqrt{\frac{1}{\lambda_i}}$.
    The resulting vector $z'_j$ is in affine-invariant coordinates.
4.  **Define Splitting Hyperplane:**
    Define a splitting hyperplane in the affine-invariant coordinate system (e.g., using PCA splitting or bisector methods). Let the hyperplane be defined by normal $n'$ and offset $b'$ such that $n'^T z' + b' = 0$.
5.  **Transform Hyperplane Back:**
    Transform the hyperplane parameters back to the original coordinate system.
    a. The normal in original coordinates is $n = (T^{-1})^T n'$, where $T$ is the matrix with columns $\sqrt{\lambda_i} e_i$.
    b. The offset $b$ is adjusted to maintain the plane equation in original coordinates.
6.  **Assign Points:** Assign points to clusters based on their position relative to the transformed hyperplane.

**Parameters:**
*   Eigenvalue threshold: Not given in the sources. Small eigenvalues may cause numerical instability when taking $\sqrt{1/\lambda_i}$ [G].

**Edge Cases and Pitfalls:**
*   **Recursive Splitting:** The method may fail on data with recursive splitting along similar directions, leading to poor approximation behavior in some regions [S1, unofficial].
*   **Smoothness Requirement:** It works well on densely sampled, smooth data sets but may not be optimal for sparse or highly irregular data [S1, unofficial].
*   **Function Values:** The basic affine-invariant approach does not consider associated function values, which may lead to suboptimal cluster shapes for scalar field approximation compared to error-based splitting [S1, unofficial].

## Subtopics

### Covariance Matrix Clustering
A clustering update step that results in normalized covariance matrices as cluster centroids, ensuring the centroid itself is in a normalized form [S2]. It ensures that the scaling vector of the centroid has unit norm, derived from the trace of the normalized covariance matrix [S2]. The formula for the squared norm of the scaling vector $s_c$ of the centroid $\hat{\Sigma}_c$ is:
$$ \|s_c\|_2^2 = \text{Tr}(\hat{\Sigma}_c) = \frac{\sum_{i=1}^N w_i \text{Tr}(\hat{\Sigma}_i)}{\sum_{i=1}^N w_i} = \frac{\sum_{i=1}^N w_i \|s_i\|_2^2}{\sum_{i=1}^N w_i} = 1 $$
where $w_i$ are weighting factors and $\hat{\Sigma}_i$ are normalized covariance matrices with $\|s_i\|_2 = 1$ [S2].

## Operating characteristics
*   **Time Complexity:** Dominated by covariance matrix computation ($O(Nd^2)$) and eigen-decomposition ($O(d^3)$), where $N$ is the number of points and $d$ is the dimensionality.
*   **Memory Complexity:** $O(Nd)$ to store points and $O(d^2)$ for covariance matrices.
*   **Strengths:** Invariant under affine transformations, producing well-shaped clusters even under shear [S1, unofficial].
*   **Limitations:** Poor approximation in regions with recursive splitting along similar directions; does not inherently account for function values [S1, unofficial].
*   **When to Use:** Densely sampled, smooth data sets where affine invariance is critical [S1, unofficial].
*   **When Not to Use:** Sparse data, data with complex recursive structures, or when function value awareness is required for cluster shape [S1, unofficial].

## Background
The affine-invariant approach was introduced to address the lack of shear invariance in previous clustering schemes [S1, unofficial]. It builds on the concept of an affine-invariant norm discussed in [102] (not cited in provided sources) and applies it to clustering by transforming points into a PCA-based coordinate system [S1, unofficial]. It is used in adaptive clustering for scattered points in scientific visualization [S1, unofficial].

## Sources
[S1] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
[S2] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
