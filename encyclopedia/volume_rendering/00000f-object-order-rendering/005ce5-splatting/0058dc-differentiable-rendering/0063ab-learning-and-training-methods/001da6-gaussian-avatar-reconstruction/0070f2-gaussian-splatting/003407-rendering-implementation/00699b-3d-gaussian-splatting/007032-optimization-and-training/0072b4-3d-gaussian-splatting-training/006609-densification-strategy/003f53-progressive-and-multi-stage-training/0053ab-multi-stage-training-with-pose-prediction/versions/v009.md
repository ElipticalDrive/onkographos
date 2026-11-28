# Multi-stage Training with Pose Prediction

Also called: FineSplat training phases, Training and Inference Protocol, Two-stage Training with Pose-free Inference

## Overview
Multi-stage training with pose prediction decomposes the optimization of feed-forward 3D Gaussian Splatting models into sequential phases to decouple geometry estimation from appearance refinement and to stabilize pose learning [S1, S2, unofficial]. Unlike single-stage training, this method isolates camera pose prediction from Gaussian attribute updates, preventing error propagation where incorrect poses corrupt the learned Gaussian distribution [S1, S2, unofficial]. It sets itself apart by explicitly freezing pose-dependent parameters during appearance refinement stages and utilizing a final end-to-end phase to recover pose-free inference capabilities [S1, S2, unofficial].

## How it works
The method operates on a set of Gaussian primitives $\mathcal{G}$ defined in [Densification Strategy](6234-densification-strategy.md). The core mechanism involves partitioning optimization into distinct stages, modifying trainable parameters, supervision signals, and densification triggers to transition from a coarse representation to a fine-grained one.

### Inputs and Outputs
*   **Inputs:** Input RGB images $I_{in}$, ground-truth camera poses/intrinsics (for supervised stages), and target view images $I_{gt}$ (for reconstruction loss).
*   **Outputs:** A set of 3D Gaussians $\mathcal{G}$ (positions $\mathbf{p}$, covariances $\mathbf{\Sigma}$, opacities $\alpha$, spherical harmonics coefficients $\mathbf{c}$) and predicted camera poses/intrinsics for target views.

### Data Structures
The model typically consists of:
1.  **Geometry/Reconstructor Branch:** Predicts camera poses and initial Gaussian centers/attributes.
2.  **Attribute/Refiner Branch:** Refines Gaussian attributes (SH, opacity, scale) given the geometry.
3.  **Gaussian Decoder:** Converts latent features or anchor points into final Gaussian parameters.

### Numbered Steps

**Stage 1: Coarse Geometry and Pose Learning**
1.  Initialize the model with a pixel-aligned Gaussian prediction head.
2.  Train for $N_1$ iterations (e.g., 120K [S1, unofficial] or until convergence).
3.  Input: RGB images and ground-truth poses.
4.  Supervision: Masked L1 and LPIPS losses on input and sampled target renderings [S2, unofficial]. Camera losses on predicted poses/intrinsics [S1, unofficial].
5.  Update: Only the attribute branch’s last $K$ Transformer blocks and the Gaussian decoder [S2, unofficial], or the full pixel-aligned decoder [S1, unofficial].
6.  Transition: Replace the pixel-aligned decoder with a patch-aligned Gaussian decoder [S1, unofficial].

**Stage 2: Appearance Refinement with Fixed Geometry**
1.  Freeze the Stage 1 trained parameters (specifically the geometry/pose estimator) [S1, S2, unofficial].
2.  Train for $N_2$ iterations (e.g., 80K [S1, unofficial] or 20K [S1, unofficial]).
3.  Input: RGB images and **ground-truth** camera poses/intrinsics.
4.  Supervision: Render target views at every step $i \in \{0, \dots, T-1\}$ (where $T=4$ [S1, unofficial]). Compute loss averaged over steps:
    $$ \mathcal{L} = \frac{1}{T} \sum_{i=0}^{T-1} \left( \mathcal{L}_2(\hat{I}^{(i)}, I) + \lambda \mathcal{L}_{perc}(\hat{I}^{(i)}, I) \right) $$
    where $\mathcal{L}_2$ is mean squared error, $\mathcal{L}_{perc}$ is VGG-19 perceptual loss, and $\lambda = 0.2$ [S1, unofficial].
5.  Detach tokens between steps so each step learns to improve the state it receives independently [S1, unofficial].
6.  Update: Only the attribute/refiner branch [S2, unofficial] or the patch-aligned decoder [S1, unofficial].

**Stage 3: End-to-End Fine-Tuning (Pose-Free)**
1.  Unfreeze all parameters, including the camera pose estimator and Gaussian head [S1, unofficial].
2.  Train for $N_3$ iterations (e.g., 10K [S1, unofficial]).
3.  Input: RGB images. **No ground-truth target cameras** are used.
4.  Supervision: Predict context and target cameras. Supervise Stage-1 Gaussians. Loss averaged over steps $i=0, \dots, T$ [S1, unofficial]. No camera loss is applied [S1, unofficial].
5.  Update: Full model [S1, unofficial].

**Inference (Pose-Free)**
1.  Use the geometry branch’s predicted cameras [S2, unofficial].
2.  Normalize translations by $s_{norm}$ (derived from context camera pairwise distances or estimated depth) [S1, S2, unofficial].
3.  Render input views for refinement feedback if a refiner is present [S2, unofficial].
4.  The refiner updates scene attributes while network weights remain fixed [S2, unofficial].

### Parameters
*   **Optimizer:** AdamW with $\beta_1=0.9, \beta_2=0.95$, weight decay 0.05 (multi-dimensional params only), gradient clipping max norm 1.0, bfloat16 mixed precision [S1, unofficial].
*   **Stage 1 LR:** $2 \times 10^{-5}$, no warm-up, cosine decay to zero [S1, unofficial].
*   **Stage 2 LR:** $1 \times 10^{-4}$, 200-iteration linear warm-up, constant until 80%, cosine decay to zero over remaining 20% [S1, unofficial].
*   **End-to-End LR:** $2 \times 10^{-5}$ for all parameters, 200-iteration linear warm-up, cosine decay to zero [S1, unofficial].
*   **Perceptual Loss Weight:** $\lambda = 0.2$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Pose Error Propagation:** If Stage 2 is not trained with ground-truth poses, errors from Stage 1 pose prediction will corrupt the Gaussian attribute learning [S1, unofficial].
*   **Token Dependency:** In iterative refinement (Stage 2), failing to detach tokens between steps causes error accumulation across steps [S1, unofficial].
*   **Scale Normalization:** The scene scale is not fixed by input images. Methods differ in normalization: YoNoSplat uses the first context camera as origin and normalizes by max pairwise distance [S1, unofficial]; ZipSplat uses estimated depth [S1, unofficial]. InfiniSplat normalizes by $s_{norm}$ [S2, unofficial]. Implementers must ensure consistent normalization between training and inference.

## Subtopics

### RLSplat Training
RLSplat adapts multi-stage training for resource-linear budget control [S3, unofficial].
*   **Stage 1 (Base Training):** Train geometry backbone and decoder with **random allocations** for 40,000 steps. Sample budgets from $[2048, 16384V]$ where $V$ is input views. This exposes the decoder to diverse allocations [S3, unofficial].
*   **Stage 2 (Policy Training):** Add a policy head initialized to uniform probabilities. Fine-tune the entire model with policy loss $\mathcal{L}_{policy} = -J_{policy}$ for 5,000 steps. Sample budgets skewed toward the lower end [S3, unofficial].
*   **Losses:** $\mathcal{L}_{recon}$ (MSE + LPIPS), $\mathcal{L}_{cam}$ (translation, rotation, FOV), $\mathcal{L}_{proj}$ (reprojection with cheirality penalty), locality regularizers ($\mathcal{L}_{vor}, \mathcal{L}_{offset}$), and $\mathcal{L}_{policy}$ [S3, unofficial].
*   **Parameters:** $K_{anc} = K_{gs} = 16$ [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Significantly higher than single-stage training due to multiple sequential phases (e.g., 120K + 80K + 10K iterations [S1, unofficial]).
*   **Memory Complexity:** Similar to single-stage training per step, but requires storing intermediate states for iterative refinement (Stage 2) [S1, unofficial].
*   **Performance:** Improves novel-view synthesis metrics (PSNR, SSIM, LPIPS) by stabilizing pose learning and allowing focused refinement of appearance [S1, S2, unofficial].
*   **Strengths:** Decouples pose and geometry errors; enables pose-free inference; robust to initial pose inaccuracies.
*   **Limitations:** Increased training time; complex hyperparameter tuning across stages; requires careful normalization of scene scale [S1, S2, unofficial].
*   **When to use:** When feed-forward models suffer from pose instability or when high-fidelity appearance is required after coarse geometry is established.

## Background
Originating from the need to stabilize feed-forward 3D Gaussian Splatting, this method was introduced in FineSplat [S1, unofficial] and InfiniSplat v2 [S2, unofficial]. It addresses the instability of jointly optimizing camera poses and Gaussian attributes in a single pass. The approach builds on the principle that pose estimation and appearance refinement are distinct optimization problems that benefit from sequential treatment. RLSplat [S3, unofficial] extends this to include budget control via policy learning.

## Sources
[S1] FineSplat: Fine Patches, Local Groups, Detailed Splats, https://arxiv.org/abs/2610.25011, fetched 2026-10-29 (unofficial)
[S2] InfiniSplat v2: Analyzing and Improving Feed-Forward 3DGS from Any Views, https://arxiv.org/abs/2610.20341, fetched 2026-10-29 (unofficial)
[S3] RLSplat: Resource-Linear Budget Control for Feed-forward 3D Gaussian Splatting, https://arxiv.org/abs/2610.27712, fetched 2026-10-26 (unofficial)
