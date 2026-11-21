# Feature Visualization
Also called: 3D PCA Feature Visualization

## Overview
Feature Visualization maps high-dimensional learned feature vectors to RGB colors to make abstract neural representations interpretable [S1][S2, unofficial]. It solves the problem of visualizing latent spaces by projecting features onto the first three principal components, allowing semantic structures to be perceived as color fields [S1][S2, unofficial]. It differs from standard scalar transfer functions by operating on multi-dimensional feature embeddings rather than raw intensity or gradient values [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of feature vectors $\mathbf{f}_i \in \mathbb{R}^D$ associated with spatial locations (voxels, primitives, or tokens) [S1][S2, unofficial].
*   **Output:** A color image or volume where each location is colored $(R, G, B)$ based on its projection onto the principal components [S1][S2, unofficial].

**Data Structures:**
*   **Feature Matrix:** A matrix $F \in \mathbb{R}^{N \times D}$ where $N$ is the number of samples and $D$ is the feature dimension [S1][S2, unofficial].
*   **Principal Components:** A matrix $P \in \mathbb{R}^{D \times 3}$ containing the top 3 eigenvectors of the covariance matrix of $F$ [S1][S2, unofficial].

**Steps:**
1.  **Feature Collection:** Gather feature vectors $\mathbf{f}_i$ for all relevant spatial elements. In NeRF contexts, these are often teacher network feature maps from training views [S1]. In Gaussian Splatting contexts, these are propagated token features from a frozen encoder [S2, unofficial].
2.  **PCA Computation:** Compute the 3-dimensional Principal Component Analysis (PCA) components of the collected features [S1][S2, unofficial].
    *   Let $\mu$ be the mean of the feature vectors.
    *   Compute the covariance matrix $C = \frac{1}{N-1} \sum_{i=1}^{N} (\mathbf{f}_i - \mu)(\mathbf{f}_i - \mu)^T$.
    *   Extract the top 3 eigenvectors $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ corresponding to the largest eigenvalues [S1][S2, unofficial].
3.  **Projection:** Project each feature vector $\mathbf{f}_i$ onto the principal components to obtain 3D coordinates:
    $$ p_{i,1} = \mathbf{f}_i \cdot \mathbf{v}_1 $$
    $$ p_{i,2} = \mathbf{f}_i \cdot \mathbf{v}_2 $$
    $$ p_{i,3} = \mathbf{f}_i \cdot \mathbf{v}_3 $$
    [S1][S2, unofficial]
4.  **Normalization and Outlier Removal:**
    *   Identify outliers in the projected values (e.g., using statistical thresholds or min-max bounds) and remove them to prevent color saturation [S1].
    *   Normalize the remaining projected values to the range $[0, 1]$ using min-max scaling:
        $$ R_i = \frac{p_{i,1} - \min(p_1)}{\max(p_1) - \min(p_1)} $$
        $$ G_i = \frac{p_{i,2} - \min(p_2)}{\max(p_2) - \min(p_2)} $$
        $$ B_i = \frac{p_{i,3} - \min(p_3)}{\max(p_3) - \min(p_3)} $$
    [S1][S2, unofficial]
5.  **Rendering:** Assign the normalized values $(R_i, G_i, B_i)$ to the color channels of the corresponding spatial elements [S1][S2, unofficial].

**Parameters:**
*   **Number of Components:** Fixed at 3 for RGB visualization [S1][S2, unofficial].
*   **Outlier Removal:** Method not specified in detail, but explicitly mentioned as necessary [S1].

**Edge Cases and Pitfalls:**
*   **Independence of PCA:** PCA is fitted independently for each method-object pair or scene. Consequently, absolute colors are not comparable across different panels or objects [S2, unofficial].
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
    # Sort by descending eigenvalue
    idx = np.argsort(eigenvalues)[::-1]
    top_3_eigenvectors = eigenvectors[:, idx[:3]]
    
    # Step 3: Project features
    projections = np.dot(centered, top_3_eigenvectors) # Shape (N, 3)
    
    # Step 4: Outlier removal and normalization
    # Simple min-max normalization. Outlier removal is implied by source [S1]
    # but specific algorithm is not given. We assume standard min-max on all data
    # or after filtering. Here we apply standard min-max.
    min_vals = np.min(projections, axis=0)
    max_vals = np.max(projections, axis=0)
    
    # Avoid division by zero
    range_vals = max_vals - min_vals
    range_vals[range_vals == 0] = 1
    
    normalized = (projections - min_vals) / range_vals
    
    # Clip to [0, 1] to handle any numerical issues or implicit outlier clipping
    colors = np.clip(normalized, 0.0, 1.0)
    
    return colors
```

## Subtopics

### PCA Feature Fields Visualization
A variant used in 3D Gaussian Splatting contexts (e.g., Gaussian-JEPA) where frozen encoder token features are propagated to input primitives [S2, unofficial]. PCA is fitted independently for each method-object pair, and the first three principal components are mapped to RGB [S2, unofficial]. It is chosen when visualizing coherent feature fields over extended structures while preserving variation between object parts [S2, unofficial]. The key pitfall is that absolute colors are not comparable across different method-object pairs [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by PCA computation, $O(N \cdot D^2)$ or $O(N \cdot D)$ with iterative methods, where $N$ is the number of samples and $D$ is the feature dimension [S1][S2, unofficial].
*   **Memory Complexity:** $O(N \cdot D)$ to store features and $O(D^2)$ for the covariance matrix [S1][S2, unofficial].
*   **Strengths:** Provides an intuitive, color-based overview of high-dimensional semantic structures [S1][S2, unofficial]. Effective for comparing semantic resolution between different models (e.g., LSeg vs. DINO) [S1].
*   **Limitations:** Colors are relative to the specific dataset/scene and not globally comparable [S2, unofficial]. Information loss due to dimensionality reduction from $D$ to 3 [S1].
*   **When to use:** Exploratory analysis of learned feature spaces, debugging feature consistency, or qualitative comparison of model semantics [S1][S2, unofficial].
*   **When not to use:** When quantitative, globally consistent color mapping is required [S2, unofficial].

## Background
Originating from the need to interpret deep learning features, this method applies Principal Component Analysis (PCA) to visualize high-dimensional data [S1]. It has been used to analyze NeRF feature fields (e.g., LSeg, DINO) and more recently in 3D Gaussian Splatting (Gaussian-JEPA) [S1][S2, unofficial]. The technique relies on standard linear algebra tools like `sklearn.decomposition.PCA` [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] Gaussian-JEPA: Joint-Embedding Predictive Learning for 3D Gaussian Splats, https://arxiv.org/abs/2608.15651v1, fetched 2026-10-26 (unofficial)
