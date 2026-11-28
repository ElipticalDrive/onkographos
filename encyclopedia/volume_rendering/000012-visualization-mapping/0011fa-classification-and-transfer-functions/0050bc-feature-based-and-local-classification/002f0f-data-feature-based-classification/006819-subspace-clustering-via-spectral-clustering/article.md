# Subspace Clustering via Spectral Clustering

Also called: Spectral Subspace Clustering, Subspace Clustering

## Overview
Subspace Clustering via Spectral Clustering is a feature-based classification method that identifies low-dimensional linear subspaces within high-dimensional attribute spaces to group volume samples. It solves the problem of segmenting complex, multivariate data where samples belong to distinct structural regimes (e.g., different materials or physical phenomena) that are not separable by simple scalar thresholds. Unlike [Similarity-based Classification](12242-similarity-based-classification.md) which relies on pairwise distances, this method exploits the geometric property that samples from the same subspace can be linearly reconstructed from one another, revealing cluster structure through algebraic relationships rather than metric proximity.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $T$ data samples $\{\mathbf{x}_i \in \mathbb{R}^D\}_{i=1}^T$, where each $\mathbf{x}_i$ is a feature vector (e.g., intensity, gradient, color) associated with a spatial position in the volume.
*   **Output:** A clustering assignment for each sample, identifying which linear subspace it belongs to. This assignment drives the transfer function design, mapping subspace labels to opacity and color values.

**Data Structures:**
1.  **Data Matrix $X$:** A $D \times T$ matrix where each column is a sample $\mathbf{x}_i$.
2.  **Coefficient Matrix $W$:** A $T \times T$ matrix where $W_{ij}$ represents the weight of sample $\mathbf{x}_j$ in the reconstruction of sample $\mathbf{x}_i$.
3.  **Affinity Matrix $A$:** A symmetric $T \times T$ matrix used for spectral clustering, derived from $W$.

**Algorithm Steps:**

1.  **Assumption:** Assume the data samples are drawn from a union of $n \ge 1$ linear subspaces $\{S_j\}_{j=1}^n$ with unknown dimensions $0 < d_j < D$ [S1].

2.  **Coefficient Matrix Construction:**
    Construct the coefficient matrix $W$ such that the data is approximately reconstructed by itself:
    $$ X \approx XW $$
    Subject to the constraint:
    $$ W_{ii} = 0 \quad \text{for } i = 1, \dots, T $$
    This constraint ensures that a sample is not used for its own reconstruction [S1]. The problem is highly ill-posed; therefore, an appropriate regularization penalty must be applied to $W$. Common choices include:
    *   Sparsity (L1 norm) [S1]
    *   Low-rank constraint [S1]
    *   L2 norm regularization [S1]
    
    *Note: The specific optimization solver is not detailed in the source, but the formulation requires solving for $W$ under the chosen regularization [S1].*

3.  **Affinity Matrix Construction:**
    Compute the adjacency (affinity) matrix $A$ from $W$:
    $$ A = |W| + |W^T| $$
    Here, $|W|$ denotes the element-wise absolute value of $W$. This step creates a symmetric graph where edges represent the linear dependency between samples [S1].

4.  **Spectral Clustering:**
    Perform spectral clustering on the graph defined by $A$ to segment the data into distinct subspaces. The source notes that "Any suitable clustering... can be used," but specifies spectral clustering as the effective method for revealing subspace structure [S1].

5.  **Subspace Fitting (Post-Clustering):**
    Once samples are segmented into clusters, fit a linear model (e.g., via PCA) to each subspace to estimate its dimension and basis. This step is crucial for understanding the transitions between subspace views in dynamic projections [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def subspace_clustering_spectral(X, regularization_type='sparsity'):
    """
    X: numpy array of shape (D, T), D features, T samples.
    regularization_type: string, one of 'sparsity', 'low_rank', 'L2'.
    Returns: cluster_labels array of shape (T,)
    """
    T = X.shape[1]
    
    # Step 1 & 2: Construct Coefficient Matrix W
    # Solve X = X * W subject to W_ii = 0 and regularization
    # This is an optimization problem. 
    # For sparsity: min ||W||_1 s.t. X = XW, diag(W)=0
    # For L2: min ||X - XW||_F^2 + lambda * ||W||_F^2 s.t. diag(W)=0
    # For low-rank: min rank(W) s.t. X = XW, diag(W)=0
    
    W = solve_coefficient_matrix(X, regularization_type)
    
    # Step 3: Construct Affinity Matrix
    # A = |W| + |W^T|
    A = abs(W) + abs(W.T)
    
    # Step 4: Spectral Clustering
    # Standard spectral clustering involves:
    # 1. Compute Laplacian L = D - A (where D is degree matrix)
    # 2. Compute eigenvectors of L corresponding to smallest eigenvalues
    # 3. Normalize rows of eigenvector matrix
    # 4. Cluster rows using k-means
    
    # The number of clusters 'n' is typically unknown. 
    # The source mentions using 3000 clusters for a specific dataset, 
    # but generally, this must be determined or provided.
    # For implementation completeness, we assume 'n_clusters' is provided.
    # If not, eigen-gap analysis on L is required.
    
    n_clusters = determine_number_of_clusters(A) # Implementation detail [G]
    
    cluster_labels = spectral_cluster(A, n_clusters)
    
    return cluster_labels

def solve_coefficient_matrix(X, reg_type):
    """
    Placeholder for the specific optimization solver.
    The source states the problem is ill-posed and requires regularization.
    """
    # Implementation depends on the chosen regularization.
    # Example for L2 with diagonal constraint:
    # W = (X^T X + lambda I)^-1 X^T X - lambda * diag(...)
    # However, the exact solver is not specified in [S1].
    raise NotImplementedError("Specific optimization solver for W is not defined in source.")

def determine_number_of_clusters(A):
    """
    Heuristic to determine n.
    Source uses 3000 for a specific case, but this is not a general rule.
    """
    # Not given in the sources.
    return 10 # Placeholder [G]

def spectral_cluster(A, n_clusters):
    """
    Standard spectral clustering algorithm.
    """
    import numpy as np
    from sklearn.cluster import KMeans
    
    # Compute Degree Matrix
    D = np.diag(np.sum(A, axis=1))
    
    # Compute Laplacian
    L = D - A
    
    # Compute Eigenvectors
    # Get the n_clusters eigenvectors corresponding to the smallest eigenvalues
    eigenvalues, eigenvectors = np.linalg.eigh(L)
    U = eigenvectors[:, :n_clusters]
    
    # Normalize rows
    row_norms = np.linalg.norm(U, axis=1, keepdims=True)
    U_normalized = U / row_norms
    
    # K-means clustering on the rows
    kmeans = KMeans(n_clusters=n_clusters, random_state=42)
    labels = kmeans.fit_predict(U_normalized)
    
    return labels
```

**Parameters:**
*   **Regularization Type:** Determines how the ill-posed reconstruction problem is solved. Options cited: Sparsity, Low-rank, L2 [S1].
*   **Number of Clusters:** The source example uses 3000 clusters for a hyperspectral dataset, but this is dataset-specific. In general, this is a hyperparameter [S1].

**Edge Cases and Pitfalls:**
*   **Ill-posedness:** The reconstruction $X \approx XW$ has multiple solutions without regularization [S1].
*   **Scalability:** The method requires constructing an $m \times m$ affinity matrix, leading to high memory and time costs for large datasets [S1].
*   **Subspace Dimension:** The dimensions of the subspaces are unknown and must be estimated post-clustering (e.g., via PCA), which may not fully exploit relationships between subspaces [S1].

## Operating characteristics
*   **Time Complexity:** $O(m^2 d)$ to construct the affinity matrix, where $m$ is the number of samples and $d$ is the data dimension [S1].
*   **Memory Complexity:** $O(m^2)$ to store the affinity matrix [S1].
*   **Strengths:** Effectively handles high-dimensional attribute spaces by exploiting linear dependencies; suitable for dynamic projections and transfer function design in multivariate visualization [S1].
*   **Limitations:** High computational cost limits the number of samples that can be processed; requires data reduction (sampling) for large volumetric datasets to terminate in reasonable time [S1].
*   **When to Use:** When data exhibits distinct linear structures in feature space (e.g., different physical materials with distinct spectral signatures) and scalar-based classification fails.
*   **When Not to Use:** For very large datasets where $O(m^2)$ memory is prohibitive, or when the data does not conform to linear subspace assumptions.

## Background
*   **Origin:** The method is described in the context of multivariate volume visualization through dynamic projections [S1].
*   **History:** It adapts techniques from machine learning literature on subspace clustering, specifically spectral clustering methods, to the domain of volume rendering and transfer function design [S1].
*   **Usage:** Used to find low-dimensional linear subspaces in the attribute space for dynamic projections and transfer function design. It allows for the identification of meaningful regions (e.g., water, urban, vegetation) in high-dimensional data like hyperspectral images [S1].

## Sources
[S1] Multivariate volume visualization through dynamic projections, https://doi.org/10.1109/ldav.2014.7013202, fetched 2026-10-26
