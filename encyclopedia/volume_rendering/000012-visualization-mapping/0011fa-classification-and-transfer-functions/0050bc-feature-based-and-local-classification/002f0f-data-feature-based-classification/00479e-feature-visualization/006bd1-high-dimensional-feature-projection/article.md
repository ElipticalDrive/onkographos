# High-Dimensional Feature Projection

Also called: Feature Space Mapping, PCA/LLE Volume Projection

## Overview
High-Dimensional Feature Projection maps high-dimensional feature vectors, which encode shape-based distances between volumetric datasets, into a two-dimensional scatterplot for visual overview and browsing [S1]. It distinguishes itself from pairwise distance visualization techniques (such as Self-Organizing Maps or Multidimensional Scaling) by focusing on comparing a small set of reference volumes against a large set of test volumes, where global linear or local non-linear projections are more efficient than all-pairs comparisons [S1].

## How it works
**Inputs:**
1. A set of $N$ volumetric datasets (e.g., virtual mastoidectomies).
2. A set of $M$ reference or composite volumes.
3. Feature extraction method (e.g., distance field histogram) producing a feature vector $f_i$ for each volume $i$.

**Outputs:**
A 2D scatterplot where each point represents a volumetric dataset, positioned according to its high-dimensional feature similarity to the references.

**Data Structures:**
- **Feature Matrix $F$**: An $N \times D$ matrix, where $N$ is the number of test volumes and $D$ is the dimensionality of the feature space. Each row $f_i$ is the concatenated feature vector for volume $i$.
- **Projection Matrix $W$**: A $D \times 2$ matrix containing the projection vectors (eigenvectors for PCA, or embedding weights for LLE).
- **Projected Points $P$**: An $N \times 2$ matrix where each row $p_i$ is the 2D coordinate $(x_i, y_i)$ for volume $i$.

**Steps:**

1. **Feature Vector Construction**:
   For each volume $i$, compute the feature vector $f_i$ by concatenating shape-based distance metrics. These metrics typically encode the distance field histogram between volume $i$ and the reference/composite volumes [S1]. Additional features (e.g., from Sec. 3.3.2 in the source) may be appended to $f_i$ [S1].
   $$ f_i = [d_1, d_2, \dots, d_D] $$
   Where $d_j$ are the individual feature components.

2. **Projection Selection**:
   Choose between Principal Component Analysis (PCA) or Locally Linear Embedding (LLE) based on user preference or data characteristics [S1].

3. **Dimensionality Reduction**:
   Map the high-dimensional vectors $F$ to 2D space using the selected method.

   **Option A: Principal Component Analysis (PCA)**
   1. Center the data: $\bar{f} = \frac{1}{N} \sum_{i=1}^{N} f_i$.
   2. Compute the covariance matrix $C = \frac{1}{N-1} (F - \mathbf{1}\bar{f}^T)^T (F - \mathbf{1}\bar{f}^T)$.
   3. Compute the eigenvectors of $C$.
   4. Select the two eigenvectors corresponding to the largest eigenvalues. Let these be $w_1$ and $w_2$.
   5. Project each vector: $p_i = [(f_i - \bar{f}) \cdot w_1, (f_i - \bar{f}) \cdot w_2]$.

   **Option B: Locally Linear Embedding (LLE)**
   1. For each point $f_i$, find its $k$ nearest neighbors in the high-dimensional space.
   2. Compute reconstruction weights $W_{ij}$ such that $f_i \approx \sum_{j \in neighbors(i)} W_{ij} f_j$, minimizing $\sum_i ||f_i - \sum_j W_{ij} f_j||^2$ subject to $\sum_j W_{ij} = 1$ [S1].
   3. Find the 2D coordinates $y_i$ that minimize the embedding cost function $E = \sum_i ||y_i - \sum_j W_{ij} y_j||^2$ [S1].

4. **Visualization**:
   Render the points $p_i$ in a 2D scatterplot. Users can browse the dataset by inspecting the spatial distribution of volumes relative to reference points [S1].

**Parameters:**
- **Projection Method**: PCA or LLE [S1].
- **Feature Components**: The specific distance metrics and histograms used to construct $f_i$ [S1].
- **k (for LLE)**: Number of neighbors. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Choice of Algorithm**: PCA is suitable for linear structures and global comparisons. LLE captures local nonlinear structures. The source notes that PCA and LLE are well-suited for comparing a small number of references to many tests, whereas Self-Organizing Maps or Multidimensional Scaling are better for all-pairwise distance comparisons [S1].
- **Feature Selection**: The quality of the projection depends on the relevance of the appended features. The source mentions that features from Sec. 3.3.2 can be appended, but their specific effects are discussed elsewhere [S1].

## Subtopics

### [Subspace Clustering for Multivariate Volume Visualization](23013-subspace-clustering-for-multivariate-volume-visualization.md)
This method is a visualization pipeline that uses subspace clustering to identify low-dimensional linear subspaces within high-dimensional multivariate volumetric data, enabling effective Transfer Function (TF) design through animated transitions (dynamic projections) between 2D views of these subspaces. It sets itself apart from static high-dimensional projection methods (like PCA) by mitigating structural illusions inherent in single-view projections; unlike random projection techniques (e.g., GGobi's grand tour), it uses data-driven subspace clustering to define meaningful transitional targets.

## Operating characteristics
**Time and Memory Complexity:**
- **PCA**: Computing the covariance matrix is $O(N D^2)$ or $O(N^2 D)$ depending on implementation. Eigendecomposition is $O(D^3)$. Projection is $O(N D)$.
- **LLE**: Finding neighbors is $O(N D \log N)$ or $O(N^2 D)$. Solving for weights is $O(N k^3)$. Solving for embedding is $O(N^3)$ or less with sparse solvers.
- **Memory**: $O(N D)$ to store the feature matrix.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- **Strengths**: Efficient overview of large-scale datasets compared to flat lists. PCA and LLE are well-suited for comparing a small number of references to a large number of test volumes [S1].
- **Limitations**: Single-view projections may suffer from structural illusions (addressed by subspace clustering variants). PCA assumes linear relationships. LLE is computationally more expensive and sensitive to parameter $k$ [S1].

**When to Use:**
- When browsing a large dataset of volumes against a few reference/composite volumes.
- When shape-based distance features are available.

**When Not to Use:**
- When comparing all pairwise distances in a set (use Self-Organizing Maps or Multidimensional Scaling instead) [S1].

## Background
The method was introduced in the context of virtual mastoidectomy performance evaluation, providing a large-scale dataset overview tool [S1]. It builds on distance field histogram methods for feature extraction [S1]. The approach leverages established dimensionality reduction techniques (PCA and LLE) adapted for volumetric shape comparison [S1].

## Sources
[S1] Virtual mastoidectomy performance evaluation through multi-volume analysis., https://doi.org/10.1007/s11548-012-0687-4, fetched 2026-11-08
