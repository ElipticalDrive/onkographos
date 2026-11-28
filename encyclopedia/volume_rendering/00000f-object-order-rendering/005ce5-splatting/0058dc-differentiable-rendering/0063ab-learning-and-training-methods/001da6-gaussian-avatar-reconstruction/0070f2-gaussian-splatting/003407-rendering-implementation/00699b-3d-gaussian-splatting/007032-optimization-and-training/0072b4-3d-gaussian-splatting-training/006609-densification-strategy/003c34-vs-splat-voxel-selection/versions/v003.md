# VS-Splat Voxel Selection
Also called: Ours, VS-Splat selection

## Overview
VS-Splat voxel selection is a learnable, differentiable mechanism for filtering 3D Gaussian primitives based on their relevance to the target object, replacing static opacity thresholding. It solves the problem of background noise and redundant primitives in sparse-view reconstruction by predicting confidence scores via Gumbel-Sigmoid and using them to refine opacity values [S1, unofficial]. It sets itself apart from siblings like [Score-based Densification](4705-score-based-densification.md) by operating on a voxel grid with learned confidence rather than gradient norms, and from [Adaptive Density Control](9188-adaptive-density-control.md) by using a stochastic binary approximation for selection rather than deterministic splitting rules.

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid (e.g., resolution $H=W=D=48$) containing fine-grained primitives (e.g., $M=16$ per voxel) [S1, unofficial].
*   **Output:** A binary mask of selected voxels and refined opacities for rendering.

**Data Structures:**
*   **Voxel Grid:** A 3D grid where each cell holds features and primitive parameters.
*   **Confidence Scores:** A scalar value per voxel, predicted by a network.
*   **Gumbel-Sigmoid Temperature ($\tau$):** A scalar parameter controlling the stochasticity of the selection.

**Steps:**
1.  **Feature Extraction:** Extract 2D feature maps from input images using a pre-trained encoder (e.g., DINOv1) [S1, unofficial].
2.  **Compression:** Compress features using a 3D convolutional compressor $C$, reducing channels from $2C$ to $\tilde{C}$ (e.g., 1536 to 128) [S1, unofficial].
3.  **Confidence Prediction:** Pass compressed features through a Selective Feature Refinement (SFR) module (containing residual convolutional blocks) to predict confidence scores for each voxel [S1, unofficial].
4.  **Stochastic Selection:** Apply the Gumbel-Sigmoid function to the confidence scores to generate a differentiable binary mask. The Gumbel-Sigmoid is defined as:
    $$ \text{GumbelSigmoid}(x, \tau) = \frac{\exp((x + \epsilon_1)/\tau)}{\exp((x + \epsilon_1)/\tau) + \exp((\epsilon_2)/\tau)} $$
    where $\epsilon_1, \epsilon_2$ are Gumbel noise samples and $\tau$ is the temperature [S1, unofficial].
5.  **Opacity Refinement:** Use the selected mask to refine the opacity values of the primitives. Voxels with low confidence are suppressed, effectively pruning background noise [S1, unofficial].
6.  **Rendering:** Render the scene using only the selected voxels/primitives.

**Parameters:**
*   **Temperature ($\tau$):** Linearly annealed from 1.0 to 0.1 over the first 50,000 iterations, then fixed at 0.1 for remaining training and inference [S1, unofficial].
*   **Voxel Resolution:** $H=W=D=48$ [S1, unofficial].
*   **Primitives per Voxel:** $M=16$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Deterministic Sigmoid Failure:** Using a standard deterministic Sigmoid instead of Gumbel-Sigmoid fails to properly select object-centric voxels, leading to significant quality drops (PSNR 18.01 vs 21.31) and excessive pruning (2,223 voxels vs 19,587) [S1, unofficial].
*   **STE Limitations:** Using a Straight-Through Estimator (STE) for gradient flow results in selecting an extremely limited number of voxels (121 voxels) and poor quality (PSNR 14.71) [S1, unofficial].
*   **Static Thresholding:** Simple opacity-based thresholding (OBT) requires manual tuning (e.g., 0.005) and is less effective at selecting object-centric voxels compared to the learnable approach [S1, unofficial].

## Subtopics

### Deterministic Sigmoid selection
A variant replacing Gumbel-Sigmoid with the deterministic Sigmoid function. It fails to select object-centric voxels properly, leading to significant rendering quality drops (PSNR 18.01, SSIM 0.835, LPIPS 0.288) and using only 2,223 voxels on the CO3D dataset [S1, unofficial].

## Operating characteristics
*   **Performance:** On the CO3D dataset, VS-Splat achieves PSNR 21.31, SSIM 0.863, LPIPS 0.213, using 19,587 voxels [S1, unofficial].
*   **Comparison:** Uses approximately 15% fewer voxels than Opacity-Based Thresholding (OBT) while outperforming it across all metrics (OBT: PSNR 21.21, SSIM 0.859, LPIPS 0.223, 22,867 voxels) [S1, unofficial].
*   **Strengths:** Learnable selection adapts to object-centric features better than static thresholds; Gumbel-Sigmoid allows for differentiable training of the selection mask.
*   **Limitations:** Requires careful tuning of the temperature schedule; deterministic alternatives perform poorly.

## Background
VS-Splat was proposed in "VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views" [S1, unofficial]. It addresses the limitations of static voxel selection methods in 3D Gaussian Splatting, particularly in sparse-view scenarios where background noise and redundant primitives degrade reconstruction quality. The method integrates a Selective Feature Refinement (SFR) module to predict voxel confidence scores, enabling end-to-end optimization of voxel selection.

## Sources
[S1] VS-Splat: Voxel-Selective feed-forward Gaussian Splatting for end-to-end 3D object reconstruction from sparse-views, https://doi.org/10.48550/arxiv.2609.12343, fetched 2026-10-26 (unofficial)
