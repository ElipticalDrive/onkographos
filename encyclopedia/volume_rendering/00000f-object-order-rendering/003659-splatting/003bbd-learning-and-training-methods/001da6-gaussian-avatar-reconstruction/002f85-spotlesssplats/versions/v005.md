# SpotlessSplats


## Overview
SpotlessSplats is a robust Gaussian Avatar Reconstruction method designed to improve reconstruction quality in casual captures by ignoring transient distractors [S1][S2, unofficial]. It enhances robustness by detecting outliers in a richer, pre-trained feature space [S1][S2, unofficial]. Unlike standard Gaussian Splatting which may be sensitive to noise, SpotlessSplats builds on [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by integrating a feature-based outlier detection mechanism to filter out transient effects and occluders [S1][S2, unofficial].

## How it works
Inputs: A set of input images or video frames of a dynamic subject, along with corresponding camera poses and parametric model parameters (e.g., SMPL pose and shape) [S2, unofficial].
Outputs: A set of Gaussians representing the avatar's geometry and appearance, optimized to be robust against transient artifacts.

Data Structures:
- Gaussian Primitives: Each Gaussian $i$ is defined by position $\mathbf{p}_i$, covariance matrix $\mathbf{\Sigma}_i$ (or scale $\mathbf{s}_i$ and rotation $\mathbf{q}_i$), opacity $\alpha_i$, and color coefficients $\mathbf{c}_i$.
- Feature Embeddings: Each Gaussian is associated with a feature vector $\mathbf{f}_i$ extracted from a pre-trained feature space.

Numbered Steps:
1. **Initialization**: Initialize Gaussians based on the parametric model (e.g., SMPL) surface points or camera positions, following the principles of [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) [S2, unofficial].
2. **Feature Extraction**: For each Gaussian, extract a feature embedding $\mathbf{f}_i$ using a pre-trained feature extractor. The specific architecture of the pre-trained model is not detailed in the provided sources [S1][S2, unofficial].
3. **Outlier Detection**: Detect outliers by analyzing the feature embeddings $\mathbf{f}_i$ in the pre-trained feature space. Gaussians identified as outliers correspond to transient distractors or occluders [S1][S2, unofficial].
4. **Filtering**: Ignore or suppress the contribution of detected outlier Gaussians during the rendering process. This is achieved by setting their opacity $\alpha_i$ to zero or excluding them from the splatting accumulation [S1][S2, unofficial].
5. **Differentiable Rendering**: Render the image using the remaining Gaussians via differentiable splatting [S2, unofficial].
6. **Optimization**: Minimize the photometric loss between the rendered image and the input frame. The loss function may include terms for color difference and potentially feature consistency, though the exact loss formulation is not specified in the sources [S2, unofficial].

Formulas:
- Rendering: $C = \sum_{i \in \mathcal{N}} \alpha_i \mathbf{c}_i \prod_{j=1}^{i-1} (1 - \alpha_j)$, where $\mathcal{N}$ is the set of non-outlier Gaussians [S2, unofficial].
- Outlier Detection: Not given in the sources. The method relies on detecting outliers in the feature space, but the specific distance metric or thresholding mechanism is not provided [S1][S2, unofficial].

Parameters:
- Pre-trained Feature Space: The specific feature extractor is not named in the sources [S1][S2, unofficial].
- Outlier Threshold: Not given in the sources.

Edge Cases and Pitfalls:
- The quality of outlier detection depends on the richness and relevance of the pre-trained feature space [S1][S2, unofficial].
- If the feature space does not distinguish well between transient distractors and valid geometry, false positives may occur [S1][S2, unofficial].

## Operating characteristics
Time and Memory Complexity: Not given in the sources.
Measured Performance: Not given in the sources.
Strengths: Enhances robustness in casual captures by ignoring transient distractors [S1][S2, unofficial].
Limitations: Relies on a pre-trained feature space, which may not be available or suitable for all domains [S1][S2, unofficial].
When to use: When capturing dynamic subjects in uncontrolled environments with potential transient occlusions or distractors [S1][S2, unofficial].
When not to use: When high-quality, controlled captures are available and robustness to transients is not a primary concern [S1][S2, unofficial].

## Background
Origin: SpotlessSplats was introduced as an extension to improve robustness in Gaussian Splatting-based reconstruction [S1][S2, unofficial]. It builds on the idea of using feature spaces to identify and ignore transient effects, similar to SWAG but using a richer pre-trained feature space [S1][S2, unofficial].
History: The method is cited in recent surveys on 3D Gaussian Splatting [S1][S2, unofficial].
Usage: Used for robust reconstruction of dynamic subjects from casual captures [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
