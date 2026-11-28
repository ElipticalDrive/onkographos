# SWAG

Also called: SWAG: Unsupervised Transient Gaussian Splatting, Unsupervised Transient Gaussians

## Overview
SWAG extends 3D Gaussian Splatting (3DGS) by conditioning Gaussians on learned appearance embeddings to handle dynamic scenes [S1][S2, unofficial]. It specifically addresses the problem of transient occluders (e.g., pedestrians, vehicles) in casual photo captures by training unsupervised transient Gaussians that are explicitly ignored during rendering [S1][S2, unofficial]. Unlike sibling methods that rely on depth regularization or pre-trained feature spaces, SWAG achieves state-of-the-art results on diverse photo scenes by directly learning to suppress occluders through appearance conditioning [S1][S2, unofficial].

## How it works
SWAG builds on [3D Gaussian Splatting](52-3d-gaussian-splatting.md) and [Blur and Transient Modeling](13851-blur-and-transient-modeling.md).

**Inputs and Outputs:**
*   **Input:** A set of unposed or posed images of a dynamic scene containing transient occluders.
*   **Output:** A set of 3D Gaussians representing the static scene geometry and appearance, with transient elements effectively removed.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $i$ retains standard 3DGS attributes: position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$), opacity $\alpha_i$, and spherical harmonic coefficients for color $\mathbf{c}_i$.
*   **Appearance Embeddings:** A learned embedding vector $\mathbf{e}_i$ is associated with each Gaussian to capture appearance variations [S1][S2, unofficial].

**Method Steps:**
1.  **Initialization:** Initialize Gaussians from the input images using standard 3DGS initialization techniques (e.g., SfM points) [G].
2.  **Appearance Conditioning:** Condition the Gaussian attributes (specifically color and potentially opacity) on the learned appearance embeddings $\mathbf{e}_i$. This allows the model to distinguish between static scene components and transient variations [S1][S2, unofficial].
3.  **Unsupervised Transient Training:** During optimization, the model learns to identify and suppress transient occluders. Gaussians corresponding to transient objects are trained to have low opacity or are explicitly masked out during the rendering process, effectively ignoring them [S1][S2, unofficial].
4.  **Rendering:** Render the scene using only the Gaussians identified as part of the static background. The transient occluders are excluded from the final image [S1][S2, unofficial].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Transient vs. Dynamic:** The method assumes transients are occluders to be ignored, not dynamic objects to be reconstructed. If the goal is to reconstruct dynamic objects, this method is not suitable [S1][S2, unofficial].
*   **Precision:** The effectiveness relies on the ability of the appearance embeddings to distinguish static from transient features. Poor initialization or insufficient training data may lead to incomplete suppression of transients [G].

## Subtopics

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Yields state-of-the-art results on diverse photo scenes [S1][S2, unofficial].
*   **Strengths:** Handles transient occluders in casual captures without requiring explicit segmentation or depth supervision [S1][S2, unofficial].
*   **Limitations:** Does not reconstruct transient objects; only suitable for static scene reconstruction in the presence of transients [S1][S2, unofficial].
*   **When to use:** When the goal is to reconstruct a static scene from images containing transient occluders (e.g., pedestrians in street scenes) [S1][S2, unofficial].
*   **When not to use:** When the goal is to reconstruct dynamic objects or when transient objects are part of the scene of interest [S1][S2, unofficial].

## Background
SWAG was introduced as an extension to 3D Gaussian Splatting to address the challenge of transient occluders in casual photo captures [S1][S2, unofficial]. It builds on the principles of appearance conditioning and unsupervised learning to achieve robust static scene reconstruction [S1][S2, unofficial]. The method is part of the broader trend of improving 3DGS robustness in real-world scenarios [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
