# Overall Training Loss

Also called: Composite Training Loss, End-to-End Training Objective, Multi-term Loss Function, PDRF Training Loss

## Overview
The overall training loss is the composite objective function used to optimize radiance fields and Gaussian splatting models end-to-end. It aggregates multiple loss terms—such as photometric reconstruction, geometric consistency, and regularization—to guide the network parameters toward a solution that minimizes rendering error while enforcing structural priors. Unlike single-term objectives, it balances competing goals (e.g., pixel accuracy vs. smoothness) via weighted hyperparameters.

## How it works
The specific formulation of the overall training loss depends on the underlying representation (e.g., NeRF, 3D Gaussian Splatting, or explicit volumes) and the training protocol (single-stage vs. multi-stage). The following sections detail the most common configurations found in the literature.

### Inputs and Outputs
*   **Inputs:**
    *   Rendered images/volumes $\hat{I}$ or $\hat{V}$.
    *   Ground truth images/volumes $I$ or $V_{gt}$.
    *   Auxiliary predictions: depth maps $\hat{D}$, normals, masks $M$, point clouds.
    *   Model parameters: Gaussian attributes (position $p$, opacity $\tilde{o}$, rotation $\tilde{r}$, scale $\tilde{s}$, color $c$), neural field weights, or voxel features.
*   **Outputs:**
    *   Scalar loss value $L_{tot}$ used for backpropagation.

### Data Structures
*   **Masks:** Binary or float tensors $M$ where $M_{ij} = -1$ (or $0$) indicates background/masked-out pixels, and $M_{ij} = 1$ (or $0$) indicates foreground.
*   **Sub-volumes:** In memory-constrained training, sub-volumes $\tilde{V}$ of size $M/4 \times N/4 \times K/4$ are sampled from the full volume $V$ [S6, unofficial].

### Formulas and Steps

#### 1. Photometric and Rendering Losses
The primary term measures the difference between rendered and ground truth images. Common variants include:

*   **L1 + Perceptual/SSIM:**
    $$ \mathcal{L}_{render}(\hat{I}, I) = \mathcal{L}_1(\hat{I}, I) + \lambda \cdot \mathcal{L}_{percept}(\hat{I}, I) $$
    where $\mathcal{L}_{percept}$ is the VGG feature distance [S4].
*   **L1 + D-SSIM:**
    $$ \mathcal{L}_{render}(I, I_{gt}) = \lambda_{L1}\mathcal{L}_1(I, I_{gt}) + \lambda_{ssim}\mathcal{L}_{ssim}(I, I_{gt}) $$
    [S6, unofficial].
*   **L1 + SSIM (Weighted Sum):**
    $L_{ren}$ is the weighted sum of L1 and SSIM loss [S3].
*   **MSE (Volume):**
    $$ \mathcal{L}_{volume}(V, V_{gt}) = \sum_{x,y,z} (V(x, y, z) - V_{gt}(x, y, z))^2 $$
    [S6, unofficial].
*   **Photometric (PDRF):**
    $$ \mathcal{L}_{photo} = \| \hat{C}^* - \tilde{C} \|_2^2 + \| \hat{C} - \tilde{C} \|_2^2 $$
    for coarse and fine renderers [S1].

#### 2. Geometric and Regularization Losses
*   **Depth Smoothness:**
    $$ \mathcal{L}_{smooth}(I, \hat{D}) = |\partial_x \hat{D}| e^{-|\partial_x I|} + |\partial_y \hat{D}| e^{-|\partial_y I|} $$
    Encourages depth gradients to align with image gradients [S4].
*   **Total Variation (TV):**
    Applied to explicit feature volumes to enforce smoothness [S1].
*   **Self-Supervised Depth:**
    $\mathcal{L}_{self-depth}$ [S2].
*   **Gaussian Point Constraint (GPC):**
    Uses a Gaussian kernel function to measure distance between inter-frame point clouds, less sensitive to noise than Chamfer distance [S2].
*   **Reprojection Loss:**
    $\mathcal{L}_{reproj}$ [S2].
*   **Codebook Losses (3DGS):**
    For rotation $\tilde{r}$ and scale $\tilde{s}$, using stop-gradient ($sg$) and codebook $Z$:
    $$ L_r = \sum_{n=1}^N \| sg[\tilde{r}_n - \hat{r}_{n}^{k-1}] - Z^k[i_n^k] \|_2^2 $$
    [S3].

#### 3. Loss Masking
To prevent background pixels from contributing to the loss:
1.  Identify pixels where mask $M = -1$ (background) [S5].
2.  Multiply loss terms (normals, AO, color) by $M$ (or a binary version) so masked pixels contribute 0 [S5].

#### 4. Multi-Stage Training Objectives
*   **Stage 1 (Initial Reconstruction):**
    $$ L_{1st} = \sum_{v=1}^V \mathcal{L}_{render}(\hat{I}_v, I_v) + \alpha \cdot \sum_{i=1}^N \mathcal{L}_{smooth}(I_i, \hat{D}_i) $$
    [S4].
*   **Stage 2 (Recurrent/End-to-End):**
    Freeze initial model. Train recurrent model with exponentially weighted rendering loss:
    $$ L_{2nd} = \sum_{t=0}^{T-1} \gamma^{T-1-t} \sum_{v=1}^V \mathcal{L}_{render}(\hat{I}_v^t, I_v) $$
    [S4].
*   **Compact 3DGS Total Loss:**
    $$ L = L_{ren} + \lambda_m L_m + L_r + L_s $$
    where $L_m$ is masking loss, $L_r$ is rotation codebook loss, $L_s$ is scale codebook loss [S3].
*   **TD-NeRF Total Loss:**
    $$ L = L_{rgb} + \lambda_1 L_{self-depth} + \lambda_2 L_{GPC} + \lambda_3 L_{reproj} $$
    [S2].
*   **PDRF Total Loss:**
    $$ L_{tot} = L_{CRR\_photo} + L_{FVR\_photo} + \lambda(L_{TV\_coar} + L_{TV\_fine}) $$
    [S1].

### Parameters and Typical Values
*   $\alpha$ (Depth smoothness weight): 0.01 [S4].
*   $\lambda$ (Perceptual weight): 0.5 [S4].
*   $\lambda_m$ (Masking regularization): Hyperparameter [S3].
*   $\lambda_1, \lambda_2, \lambda_3$ (TD-NeRF weights): 0.04, 1.0, 1.0 [S2].
*   $\gamma$ (Exponential decay for recurrent steps): 0.9 [S4].
*   $\lambda_{L1}, \lambda_{ssim}$: Weights for L1 and SSIM [S6, unofficial].
*   Batch size: 1024 rays [S1].
*   Gamma correction: 2.2 (camera response function) [S1].

### Edge Cases and Pitfalls
*   **Background Matching:** Without loss masking, the network must match default ground truth values in empty regions, complicating training [S5].
*   **Memory Constraints:** Rendering with full volume resolution ($256^3$) requires exhaustive GPU memory for gradients. Solution: Randomly sample sub-volumes ($1/4$ resolution) during training [S6, unofficial].
*   **Codebook Training Cost:** R-VQ and codebook learning are computationally heavy. Apply only for the last 1K iterations with K-means initialization; set $L_r, L_s$ to zero otherwise [S3].
*   **Chamfer Sensitivity:** Standard Chamfer distance for point cloud constraints is sensitive to depth noise. Use Gaussian Point Constraint (GPC) instead [S2].

## Subtopics

### Loss Masking
A technique to modulate loss terms for normals, ambient occlusion, and colors with the output mask. Pixels where the mask $M$ is -1 (background) are excluded from the loss calculation. This simplifies the network's task by preventing it from trying to match default background values in empty regions [S5].

### X-GRM Training Objectives
Specific to X-ray Gaussian Reconstruction Models. Uses volume MSE loss and rendering L1+SSIM loss. To manage GPU memory, sub-volumes of size $M/4 \times N/4 \times K/4$ are randomly sampled from the original volume $V$ during training, while the full volume is used at inference [S6, unofficial].

## Operating characteristics
*   **Complexity:** Training is memory-intensive, particularly for explicit volumes or large Gaussian counts. Sub-volume sampling reduces memory usage by a factor of 64 ($4^3$) [S6, unofficial].
*   **Performance:** Multi-stage training (Stage 1 initialization + Stage 2 refinement) improves convergence and final quality [S4].
*   **Scalability:** Codebook-based approaches (e.g., Compact 3DGS) reduce parameter count (48 params for SH vs 59) but add complexity to the loss function [S3].
*   **Strengths:** Combines photometric accuracy with geometric priors (depth, smoothness) for robust reconstruction.
*   **Limitations:** Requires careful tuning of multiple hyperparameters ($\lambda$ weights). Some terms (like codebook losses) are computationally expensive and may need to be disabled for most of training [S3].

## Background
The concept of composite training losses in radiance fields evolved from early NeRF works combining L1 and SSIM [S1, S3] to more complex multi-stage and multi-modal objectives.
*   **PDRF:** Introduced progressive deblurring with coarse/fine photometric losses and TV regularization [S1].
*   **TD-NeRF:** Added depth priors and Gaussian point constraints for joint pose and field optimization [S2].
*   **Compact 3DGS:** Adapted 3DGS with codebook regularization for compact representation [S3].
*   **ReSplat:** Introduced two-stage training with recurrent updates and depth smoothness [S4].
*   **X-GRM:** Applied volume and rendering losses to sparse-view X-ray CT reconstruction [S6, unofficial].
*   **Volumetric Isosurface:** Introduced loss masking for deep learning-based super-resolution [S5].

## Sources
[S1] PDRF: Progressively Deblurring Radiance Field for Fast Scene Reconstruction from Blurry Images, https://doi.org/10.1609/aaai.v37i2.25295, fetched 2026-10-26
[S2] TD-NeRF: Novel Truncated Depth Prior for Joint Camera Pose and Neural Radiance Field Optimization, https://doi.org/10.1109/iros58592.2024.10802634, fetched 2026-10-26
[S3] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S4] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
[S5] Volumetric Isosurface Rendering with Deep Learning-Based Super ..., https://ieeexplore.ieee.org/abstract/document/8918030, fetched 2026-10-31
[S6] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
