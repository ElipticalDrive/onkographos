# PCA Feature Fields Visualization
Also called: Feature Map Visualization, PCA Feature Visualization

## Overview
PCA Feature Fields Visualization is a technique that maps continuous, frozen encoder features to RGB color space by projecting token features onto the first three principal components [S1, unofficial]. It solves the problem of visualizing high-dimensional latent representations in 3D scenes by reducing dimensionality while preserving structural coherence [S1, S2, unofficial]. Unlike standard RGB rendering, this method reveals semantic or geometric relationships within the feature space, distinguishing object parts from background noise [S2, unofficial]. It is distinct from uncertainty visualization, which maps scalar variance values, and from global embedding organization, which uses t-SNE for 2D scatter plots rather than spatial field rendering [S1, unofficial].

## How it works
**Inputs:**
- Frozen encoder token features propagated to input primitives (e.g., Gaussians) [S1, unofficial].
- A set of frames or views containing pixel-aligned feature vectors [S2, unofficial].

**Outputs:**
- RGB images where each pixel's color corresponds to the projected feature vector [S1, S2, unofficial].

**Data Structures:**
- **Feature Vectors:** For each primitive or pixel, a vector $\mathbf{f} \in \mathbb{R}^D$, where $D$ is the latent dimension [S1, S2, unofficial].
- **PCA Components:** A $3 \times D$ matrix $\mathbf{W}$ containing the top 3 eigenvectors of the feature covariance matrix [S1, S2, unofficial].

**Steps:**
1. **Feature Propagation:** Propagate frozen encoder token features to the input primitives (e.g., 3D Gaussians) [S1, unofficial].
2. **Collection:** Collect all pixel-aligned feature vectors across all frames or views for a specific method-object pair [S1, S2, unofficial].
3. **PCA Fitting:** Fit Principal Component Analysis independently for each method-object pair on the collected features [S1, unofficial].
   - Compute the covariance matrix of the feature vectors.
   - Extract the top 3 eigenvectors corresponding to the largest eigenvalues [S1, S2, unofficial].
4. **Projection:** Project each feature vector $\mathbf{f}_i$ onto the first three principal components:
   $$ \mathbf{p}_i = \mathbf{W} \mathbf{f}_i $$
   where $\mathbf{p}_i \in \mathbb{R}^3$ [S1, S2, unofficial].
5. **RGB Mapping:** Map the components of $\mathbf{p}_i$ to RGB channels. Typically, the three components are assigned directly to Red, Green, and Blue channels [S1, unofficial].
6. **Background Handling:** Fill background areas with low density with noise according to the feature sampling equation (Eq. 3 in [S2, unofficial]) [S1, S2, unofficial]. In latentSplat, this is consistent with setting background standard deviation to one [S2, unofficial].

**Parameters:**
- **Number of Components:** 3 (fixed for RGB visualization) [S1, S2, unofficial].
- **PCA Scope:** Independent per method-object pair [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Color Incomparability:** Absolute colors are not comparable across different panels or methods because PCA is fitted independently for each method-object pair [S1, unofficial]. Colors must be interpreted within each panel [S1, unofficial].
- **Background Noise:** Background areas with low density are filled with noise, which may appear as uniform or random patterns depending on the sampling equation [S2, unofficial].
- **Normalization:** In global embedding analysis, features are $\ell_2$-normalized before PCA, but for field visualization, direct projection is used [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pca_feature_fields_visualization(frozen_features, primitives, frames):
    """
    Visualize frozen encoder features using PCA.
    
    Args:
        frozen_features: List of feature vectors for each primitive/frame.
        primitives: List of primitives (e.g., Gaussians) with positions and densities.
        frames: List of frames/views to process.
        
    Returns:
        RGB images for each frame.
    """
    # Step 1: Collect all pixel-aligned feature vectors across all frames
    all_features = []
    for frame in frames:
        # Propagate frozen encoder token features to input primitives [S1, unofficial]
        # This step assumes features are already propagated to primitives
        # Collect features for visible primitives/pixels
        frame_features = get_pixel_aligned_features(frame, primitives)
        all_features.extend(frame_features)
    
    # Step 2: Fit PCA independently for each method-object pair [S1, unofficial]
    # Compute covariance matrix
    mean = np.mean(all_features, axis=0)
    centered_features = all_features - mean
    cov_matrix = np.cov(centered_features, rowvar=False)
    
    # Extract top 3 eigenvectors
    eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)
    # Sort by eigenvalue descending
    idx = np.argsort(eigenvalues)[::-1]
    top_3_eigenvectors = eigenvectors[:, idx[:3]]
    
    # Step 3: Project features onto first three principal components [S1, unofficial]
    projected_features = np.dot(centered_features, top_3_eigenvectors)
    
    # Step 4: Map to RGB channels [S1, unofficial]
    # Normalize projected features to [0, 1] for display
    min_val = np.min(projected_features, axis=0)
    max_val = np.max(projected_features, axis=0)
    normalized_features = (projected_features - min_val) / (max_val - min_val + 1e-8)
    
    # Assign to RGB
    rgb_images = []
    for frame_idx, frame in enumerate(frames):
        # Reconstruct frame-specific projected features
        frame_projected = projected_features[frame_start:frame_end]
        frame_rgb = np.clip(frame_projected, 0, 1)
        
        # Step 5: Handle background areas with low density [S2, unofficial]
        # Fill with noise according to feature sampling equation [S2, unofficial]
        # This is a placeholder for the specific noise generation method
        background_mask = get_low_density_mask(frame, primitives)
        frame_rgb[background_mask] = generate_noise(frame_rgb.shape)
        
        rgb_images.append(frame_rgb)
        
    return rgb_images
```

## Operating characteristics
- **Time Complexity:** Dominated by PCA fitting, which is $O(N \cdot D^2)$ or $O(N \cdot D)$ with iterative methods, where $N$ is the number of feature vectors and $D$ is the feature dimension [S1, S2, unofficial]. Projection is $O(N \cdot D)$.
- **Memory Complexity:** $O(N \cdot D)$ to store features and $O(D^2)$ for covariance matrix [S1, S2, unofficial].
- **Strengths:** Reveals coherent fields over extended structures and preserves variation between object parts [S1, unofficial]. Clearly separates background from foreground [S2, unofficial].
- **Limitations:** Colors are not comparable across different methods or objects due to independent PCA fitting [S1, unofficial]. Requires frozen encoder features, which may not be available in all pipelines [S1, S2, unofficial].
- **When to Use:** When visualizing high-dimensional latent features in 3D scenes to understand semantic or geometric structure [S1, S2, unofficial].
- **When Not to Use:** When absolute color consistency across different methods or objects is required [S1, unofficial].

## Background
- **Origin:** Introduced in Gaussian-JEPA [S1, unofficial] and latentSplat [S2, unofficial] for visualizing frozen encoder features in 3D Gaussian splats.
- **Development:** Used to analyze the quality of learned feature representations in novel view synthesis and 3D reconstruction tasks [S1, S2, unofficial].
- **Usage:** Applied in research on joint-embedding predictive learning and variational Gaussians for 3D scene representation [S1, S2, unofficial].

## Sources
[S1] Gaussian-JEPA: Joint-Embedding Predictive Learning for 3D Gaussian Splats, https://arxiv.org/abs/2608.15651v1, fetched 2026-10-26 (unofficial)
[S2] latentSplat, https://arxiv.org/abs/2403.16292, fetched 2026-10-27 (unofficial)
