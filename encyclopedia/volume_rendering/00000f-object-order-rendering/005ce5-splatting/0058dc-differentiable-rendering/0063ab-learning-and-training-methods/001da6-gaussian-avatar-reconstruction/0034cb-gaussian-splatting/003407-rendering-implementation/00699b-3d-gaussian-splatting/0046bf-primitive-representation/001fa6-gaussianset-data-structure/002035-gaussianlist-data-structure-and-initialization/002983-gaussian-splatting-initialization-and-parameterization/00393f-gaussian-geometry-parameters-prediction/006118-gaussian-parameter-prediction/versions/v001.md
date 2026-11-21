# Gaussian Parameter Prediction

Also called: 3D Gaussian Splatting Parameter Prediction

## Overview
Gaussian Parameter Prediction is the process of deriving the geometric and appearance parameters of 3D Gaussians from input data, such as images or features, to enable novel view synthesis. It solves the problem of initializing or regenerating the Gaussian set without iterative optimization. It differs from its siblings by focusing on direct prediction from pooled features or unprojected depth maps rather than iterative densification or mesh embedding.

## How it works
The method varies by source. [S1] predicts parameters from a single view using depth unprojection and a DPT head. [S2] predicts parameters from aggregated features of a Gaussian Graph.

### Inputs and Outputs
*   **Inputs:**
    *   [S1]: Input image $I_i$, per-pixel depth map $D$, camera parameters $c_i$, monocular features $F_{mono}$.
    *   [S2]: Pooled features $f$ from a Gaussian Graph aggregation.
*   **Outputs:**
    *   Gaussian centers $\mu_j$.
    *   Covariance matrices $\Sigma_j$ (derived from rotation $R$ and scale $S$).
    *   Opacity $\alpha_j$.
    *   Color $c_j$.

### Data Structures
*   **Gaussian Center:** 3D vector $\mu \in \mathbb{R}^3$.
*   **Covariance:** 3x3 symmetric positive definite matrix $\Sigma \in \mathbb{R}^{3 \times 3}$.
*   **Rotation/Scale:** Rotation matrix $R \in \mathbb{R}^{3 \times 3}$, Scale matrix $S \in \mathbb{R}^{3 \times 3}$ (diagonal).
*   **Features:** Vector $f \in \mathbb{R}^D$.

### Steps

#### Method 1: DepthSplat [S1]
1.  **Unproject Depth:** Unproject per-pixel depth maps to 3D space using camera parameters to obtain Gaussian centers $\mu_j$.
2.  **Feature Extraction:** Extract monocular features $F_{mono}$ using a pre-trained backbone (e.g., Depth Anything V2 [70]). Resize to match input resolution.
3.  **Concatenation:** Concatenate the original image, depth map, and monocular feature information.
4.  **Prediction:** Pass the concatenated input through a DPT head [41] to predict remaining parameters: opacity $\alpha_j$, covariance $\Sigma_j$, and color $c_j$.
5.  **Rendering:** Render novel views using the predicted 3D Gaussians with the Gaussian splatting operation [29].

#### Method 2: Gaussian Graph Network [S2]
1.  **Feature Aggregation:** Compute aggregated features $f$ from the Gaussian Graph using a pooling operation $\phi_{pooling}$.
    $$ (\mu, f) = \psi_{pooling}(G) $$
2.  **Parameter Prediction:** Map the pooled feature $f$ to parameters using prediction heads $\psi_R, \psi_S, \psi_\alpha, \psi_c$:
    $$ R = \phi_R(f), \quad S = \phi_S(f), \quad \alpha = \phi_\alpha(f), \quad c = \phi_c(f) $$
3.  **Covariance Construction:** Compute the covariance matrix $\Sigma$ using the predicted rotation and scale matrices:
    $$ \Sigma = R S S^\top R^\top $$
    where $R$ is the rotation matrix and $S$ is the scale matrix.

### Formulas
*   **Covariance:** $\Sigma = R S S^\top R^\top$ [S2].
*   **Feature Aggregation (Graph):**
    $$ \hat{f}_i = \sum_{j=1}^{N} \tilde{a}_{ij} E_{j \to i} f_j W \in \mathbb{R}^D $$
    where $\tilde{a}_{ij}$ are normalized attention weights, $E_{j \to i}$ is an edge feature, and $W$ is a weight matrix [S2].
*   **Pooling:**
    $$ \phi_{pooling}(G) = \bigcup_{s=1}^{l} \phi_{pooling}(G_s) $$
    where $G_s$ are connected subgraphs [S2].

### Parameters
*   **Prediction Heads:** $\phi_R, \phi_S, \phi_\alpha, \phi_c$ are neural network layers mapping $f$ to parameters [S2].
*   **Monocular Backbone:** ViT with patch size 14 [S1].

### Edge Cases and Pitfalls
*   **Precision:** Unprojection accuracy depends on depth map quality [S1].
*   **Boundaries:** Handling empty regions or reflective surfaces requires monocular features [S1].

## Subtopics

### Parameter Prediction from Pooled Features
A variant where Gaussian parameters are predicted directly from features aggregated via a Gaussian Graph pooling operation. It sets itself apart by using graph-based aggregation to capture local structure before prediction. Choose this when local geometric consistency is critical. See [Gaussian Graph Network](10060-deformable-3d-gaussian-splatting.md).

## Operating characteristics
*   **Time Complexity:** Depends on the backbone and prediction head size. [S1] uses a ViT backbone, which is computationally intensive. [S2] uses graph operations, scaling with the number of nodes and edges.
*   **Memory Complexity:** Stores features for all Gaussians. [S1] stores features per pixel. [S2] stores graph structures.
*   **Strengths:** Fast inference compared to iterative optimization. [S1] handles challenging scenes with monocular features. [S2] generalizes well across datasets.
*   **Limitations:** Quality depends on the pre-trained backbone or graph structure. [S1] may struggle with unprojected depth errors. [S2] requires graph construction.

## Background
Originated from the need for fast, non-iterative 3D Gaussian Splatting initialization. [S1] (DepthSplat) connects Gaussian Splatting with monocular depth estimation. [S2] (Gaussian Graph Network) introduces a graph-based representation for learning efficient and generalizable Gaussian representations.

## Sources
[S1] DepthSplat: Connecting Gaussian Splatting and Depth, https://doi.org/10.1109/cvpr52734.2025.01534, fetched 2026-10-26
[S2] Gaussian Graph Network: Learning Efficient and Generalizable Gaussian Representations from Multi-view Images, https://doi.org/10.48550/arXiv.2503.16338, fetched 2026-10-29
