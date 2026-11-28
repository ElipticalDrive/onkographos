# Feature Visualization
Also called: 3D PCA Feature Visualization

## Overview
Feature Visualization maps high-dimensional learned feature vectors to RGB colors to make abstract neural representations interpretable [S1]. It solves the problem of visualizing latent spaces by projecting features onto the first three principal components, allowing semantic structures to be perceived as color fields [S1]. It differs from standard scalar transfer functions by operating on multi-dimensional feature embeddings rather than raw intensity or gradient values [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of feature vectors $\mathbf{f}_i \in \mathbb{R}^D$ associated with spatial locations (voxels, primitives, or tokens) [S1].
*   **Output:** A color image or volume where each location is colored $(R, G, B)$ based on its projection onto the principal components [S1].

**Data Structures:**
*   **Feature Matrix:** A matrix $F \in \mathbb{R}^{N \times D}$ where $N$ is the number of samples and $D$ is the feature dimension [G].
*   **Principal Components:** A matrix $P \in \mathbb{R}^{D \times 3}$ containing the top 3 eigenvectors of the covariance matrix of $F$ [G].

**Steps:**
1.  **Feature Collection:** Gather feature vectors $\mathbf{f}_i$ for all relevant spatial elements. In NeRF contexts, these are teacher network feature maps from the first frame of training view images [S1].
2.  **PCA Computation:** Compute the 3-dimensional Principal Component Analysis (PCA) components of the collected features [S1].
    *   Let $\mu$ be the mean of the feature vectors [G].
    *   Compute the covariance matrix $C = \frac{1}{N-1} \sum_{i=1}^{N} (\mathbf{f}_i - \mu)(\mathbf{f}_i - \mu)^T$ [G].
    *   Extract the top 3 eigenvectors $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ corresponding to the largest eigenvalues [G].
3.  **Projection:** Project each feature vector $\mathbf{f}_i$ onto the principal components to obtain 3D coordinates:
    $$ p_{i,1} = \mathbf{f}_i \cdot \mathbf{v}_1 $$
    $$ p_{i,2} = \mathbf{f}_i \cdot \mathbf{v}_2 $$
    $$ p_{i,3} = \mathbf{f}_i \cdot \mathbf{v}_3 $$
    [G]
4.  **Normalization and Outlier Removal:** Identify outliers in the projected values and remove them to prevent color saturation [S1]. Normalize the remaining projected values to the range $[0, 1]$ using min-max scaling:
    $$ R_i = \frac{p_{i,1} - \min(p_1)}{\max(p_1) - \min(p_1)} $$
    $$ G_i = \frac{p_{i,2} - \min(p_2)}{\max(p_2) - \min(p_2)} $$
    $$ B_i = \frac{p_{i,3} - \min(p_3)}{\max(p_3) - \min(p_3)} $$
    [S1]
5.  **Rendering:** Assign the normalized values $(R_i, G_i, B_i)$ to the color channels of the corresponding spatial elements [S1].

**Parameters:**
*   **Number of Components:** Fixed at 3 for RGB visualization [S1].
*   **Outlier Removal:** Method not specified in detail, but explicitly mentioned as necessary [S1].

**Edge Cases and Pitfalls:**
*   **Independence of PCA:** PCA is fitted independently for each method-object pair or scene [S1].
*   **Dimensionality Loss:** 3D PCA cannot perfectly visualize the full feature space, potentially losing semantic information [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def feature_visualization(features):
    """
    Visualizes high-dimensional features using 3D PCA.
    
    Args:
        features: np.ndarray of shape (N, D), N samples, D dimensions.
        
    Returns:
        colors: np.ndarray of shape (N, 3), RGB values in [0, 1].
    """
    # Step 1: Compute mean and covariance
    mean = np.mean(features, axis=0)
    centered = features - mean
    cov_matrix = np.cov(centered, rowvar=False)
    
    # Step 2: Compute PCA components (top 3 eigenvectors)
    eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)
    idx = np.argsort(eigenvalues)[::-1]
    top_3_eigenvectors = eigenvectors[:, idx[:3]]
    
    # Step 3: Project features
    projections = np.dot(centered, top_3_eigenvectors) # Shape (N, 3)
    
    # Step 4: Outlier removal and normalization
    # Outlier removal algorithm not specified in sources [S1]
    # Assuming standard min-max normalization on remaining data
    min_vals = np.min(projections, axis=0)
    max_vals = np.max(projections, axis=0)
    
    range_vals = max_vals - min_vals
    range_vals[range_vals == 0] = 1
    
    normalized = (projections - min_vals) / range_vals
    
    # Clip to [0, 1] to handle numerical issues
    colors = np.clip(normalized, 0.0, 1.0)
    
    return colors
```

## Subtopics

### [PCA Feature Fields Visualization](16488-pca-feature-fields-visualization.md)
Maps continuous frozen encoder features to RGB by projecting token features onto the first three principal components. It solves the problem of visualizing high-dimensional latent representations in 3D scenes by reducing dimensionality while preserving structural coherence. Unlike standard RGB rendering, it reveals semantic or geometric relationships within the feature space, distinguishing object parts from background noise. It is distinct from uncertainty visualization and from global embedding organization using t-SNE. [PCA Feature Fields Visualization](16488-pca-feature-fields-visualization.md)

### High-Dimensional Feature Projection
Concatenates all computed feature vectors into a high-dimensional feature space encoding shape-based distances to reference or composite volumes [S2]. Maps the high-dimensional vectors to a 2D scatterplot using either Principal Component Analysis (PCA) or Locally Linear Embedding (LLE) [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides an intuitive, color-based overview of high-dimensional semantic structures [S1]. Effective for comparing semantic resolution between different models (e.g., LSeg vs. DINO) [S1].
*   **Limitations:** Colors are relative to the specific dataset/scene and not globally comparable [S1]. Information loss due to dimensionality reduction from $D$ to 3 [S1].
*   **When to use:** Exploratory analysis of learned feature spaces, debugging feature consistency, or qualitative comparison of model semantics [S1].
*   **When not to use:** When quantitative, globally consistent color mapping is required [S1].

## Background
Originating from the need to interpret deep learning features, this method applies Principal Component Analysis (PCA) to visualize high-dimensional data [S1]. It has been used to analyze NeRF feature fields (e.g., LSeg, DINO) [S1]. The technique relies on standard linear algebra tools like `sklearn.decomposition.PCA` [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] Virtual mastoidectomy performance evaluation through multi-volume analysis., https://doi.org/10.1007/s11548-012-0687-4, fetched 2026-11-08
