# Gaussian Parameter Prediction

Also called: 3D Gaussian Splatting Parameter Prediction

## Overview
Gaussian Parameter Prediction derives the geometric and appearance parameters of 3D Gaussians from input data to enable novel view synthesis [S1, S2]. It solves the problem of regenerating the Gaussian set directly from features or depth maps without iterative optimization [S1, S2]. It differs from its siblings by focusing on direct prediction from pooled features or unprojected depth maps rather than iterative densification or mesh embedding [S1, S2].

## How it works
The method varies by source. [S1] predicts parameters from a single view using depth unprojection and a DPT head. [S2] predicts parameters from aggregated features of a Gaussian Graph.

### Inputs and Outputs
*   **Inputs:**
    *   [S1]: Input image $I_i$, per-pixel depth map, camera parameters $c_i$, monocular features $F_{mono}$.
    *   [S2]: Pooled features $f$ from a Gaussian Graph aggregation.
*   **Outputs:**
    *   Gaussian centers $\mu_j$.
    *   Covariance matrices $\Sigma_j$.
    *   Opacity $\alpha_j$.
    *   Color $c_j$ [S1, S2].

### Data Structures
*   **Gaussian Center:** 3D vector $\mu \in \mathbb{R}^3$ [S1, S2].
*   **Covariance:** 3x3 symmetric positive definite matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ [S2].
*   **Rotation/Scale:** Rotation matrix $R \in \mathbb{R}^{3 \times 3}$, Scale matrix $S \in \mathbb{R}^{3 \times 3}$ (diagonal) [S2].
*   **Features:** Vector $f \in \mathbb{R}^D$ [S2].

### Steps

#### Method 1: DepthSplat [S1]
1.  **Unproject Depth:** Unproject per-pixel depth maps to 3D space using camera parameters to obtain Gaussian centers $\mu_j$ [S1].
2.  **Feature Extraction:** Extract monocular features $F_{mono}$ using a pre-trained backbone (Depth Anything V2 [70], a ViT with patch size 14) and bilinearly interpolate the spatial resolution to match the input resolution [S1].
3.  **Concatenation:** Concatenate the original image, depth map, and monocular feature information [S1].
4.  **Prediction:** Pass the concatenated input through a DPT head [41] to predict remaining parameters: opacity $\alpha_j$, covariance $\Sigma_j$, and color $c_j$ [S1].
5.  **Rendering:** Render novel views using the predicted 3D Gaussians with the Gaussian splatting operation [29] [S1].

#### Method 2: Gaussian Graph Network [S2]
1.  **Feature Aggregation:** Compute aggregated features $f$ from the Gaussian Graph using a pooling operation $\phi_{pooling}$ [S2].
2.  **Parameter Prediction:** Map the pooled feature $f$ to parameters using prediction heads $\phi_R, \phi_S, \phi_\alpha, \phi_c$ [S2].
3.  **Covariance Construction:** Compute the covariance matrix $\Sigma$ using the predicted rotation and scale matrices [S2].

### Formulas
*   **Covariance:** $\Sigma = R S S^\top R^\top$ [S2].
*   **Feature Aggregation:** $\hat{f}_i = \sum_{j=1}^{N} \tilde{a}_{ij} E_{j \to i} f_j W \in \mathbb{R}^D$, where $\tilde{a}_{ij}$ are normalized attention weights, $E_{j \to i}$ is an edge feature, and $W$ is a weight matrix [S2].
*   **Pooling:** $\phi_{pooling}(G) = \bigcup_{s=1}^{l} \phi_{pooling}(G_s)$, where $G_s$ are connected subgraphs [S2].

### Parameters
*   **Prediction Heads:** $\phi_R, \phi_S, \phi_\alpha, \phi_c$ are neural network layers mapping $f$ to parameters [S2].
*   **Monocular Backbone:** ViT with patch size 14 [S1].

### Edge Cases and Pitfalls
*   Feature-less regions and reflective surfaces are handled by integrating pre-trained monocular depth features into the cost volume [S1].
*   Details on precision, boundaries, or degenerate input handling are not given in the sources [S1, S2].

## Subtopics

### [GSGAN Hierarchical Generation](20289-gsgan-hierarchical-generation.md)
GSGAN is a generative model that synthesizes 3D Gaussian splats hierarchically to approximate complex 3D shapes and textures, building on the primitive representation defined in [Gaussian Parameter Prediction](18303-gaussian-parameter-prediction.md). It differs from standard 3D Gaussian Splatting by generating the Gaussian parameters via a deep neural network conditioned on pose and camera views, rather than optimizing them from static point clouds. This allows for the generation of novel 3D objects from 2D images. Choose this when hierarchical generative control over pose and view is required. See [GSGAN Hierarchical Generation](20289-gsgan-hierarchical-generation.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, S2].
*   **Memory Complexity:** [S1] stores features per pixel. [S2] stores graph structures.
*   **Measured Performance:** Not given in the sources [S1, S2].
*   **Scaling:** Not given in the sources [S1, S2].
*   **Strengths:** [S1] handles challenging scenes with monocular features. [S2] generalizes well across datasets.
*   **Limitations:** Not given in the sources [S1, S2].
*   **When to use:** Not given in the sources [S1, S2].

## Background
[S1] (DepthSplat) connects Gaussian Splatting with monocular depth estimation. [S2] (Gaussian Graph Network) introduces a graph-based representation for learning efficient and generalizable Gaussian representations.

## Sources
[S1] DepthSplat: Connecting Gaussian Splatting and Depth, https://doi.org/10.1109/cvpr52734.2025.01534, fetched 2026-10-26
[S2] Gaussian Graph Network: Learning Efficient and Generalizable Gaussian Representations from Multi-view Images, https://doi.org/10.48550/arXiv.2503.16338, fetched 2026-10-29
