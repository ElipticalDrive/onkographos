# Momentum-GS

Also called: Momentum Gaussian Splatting

## Overview
Momentum-GS is a block-wise large-scene training method that leverages momentum self-distillation to improve convergence quality and spatial consistency in 3D Gaussian Splatting [S1, S2, unofficial]. It addresses the limitations of distributed training in large scenes by decoupling the number of training blocks from the GPU count, using a momentum-updated teacher model to guide student optimization [S2, unofficial]. This approach restores cross-block spatial consistency and reduces the required number of blocks compared to prior methods like CityGaussian, achieving significant improvements in perceptual metrics [S2, unofficial].

## How it works
Momentum-GS operates on the principle of self-distillation within a block-wise training framework. The method divides the scene into blocks (tiles) and trains them in parallel or sequentially, but unlike standard distributed approaches, it maintains a global teacher model that aggregates information to guide local student updates [S2, unofficial].

**Inputs and Outputs:**
*   **Input:** A large-scale scene represented as a set of 3D Gaussians, divided into $B$ blocks. Initial Gaussian parameters (position, covariance, opacity, SH coefficients) for each block.
*   **Output:** Optimized Gaussian parameters for the entire scene with improved cross-block consistency.

**Data Structures:**
*   **Student Model:** The set of Gaussian parameters currently being optimized for a specific block.
*   **Teacher Model:** An exponential moving average (EMA) of the student parameters across all blocks or training steps. This acts as a global reference [S1, S2, unofficial].
*   **Block Weights:** A scalar value for each block, adjusted based on reconstruction accuracy [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize the student Gaussian parameters for each block. Initialize the teacher model as a copy of the initial student parameters or an empty state [S2, unofficial].
2.  **Training Loop:** For each training iteration:
    a. **Forward Pass (Student):** Render the current block using the student parameters. Calculate the reconstruction loss $\mathcal{L}_{rec}$ against the ground truth image [S2, unofficial].
    b. **Distillation Loss:** Render the same view using the teacher model's parameters (or the teacher's representation of the block's region). Calculate a distillation loss $\mathcal{L}_{dist}$ between the student's output and the teacher's output [S1, S2, unofficial].
    c. **Block Weighting:** Compute a weight $w_b$ for the current block based on its reconstruction accuracy (e.g., inverse of loss or a confidence metric). This weight adjusts the influence of the block's distillation loss [S2, unofficial].
    d. **Total Loss:** Combine losses: $\mathcal{L}_{total} = \mathcal{L}_{rec} + \lambda \cdot w_b \cdot \mathcal{L}_{dist}$, where $\lambda$ is a hyperparameter [S2, unofficial].
    e. **Update Student:** Update the student parameters using gradient descent on $\mathcal{L}_{total}$ [S2, unofficial].
    f. **Update Teacher:** Update the teacher model using an exponential moving average of the student parameters: $\theta_{teacher} \leftarrow \alpha \theta_{teacher} + (1-\alpha) \theta_{student}$, where $\alpha$ is the momentum coefficient (typically close to 1, e.g., 0.999) [S1, S2, unofficial].
3.  **Iteration:** Repeat until convergence.

**Formulas:**
*   **EMA Update:** $\theta_{t}^{teacher} = \alpha \theta_{t-1}^{teacher} + (1-\alpha) \theta_{t}^{student}$ [S1, S2, unofficial].
*   **Block Weighting:** $w_b = f(\text{reconstruction accuracy}_b)$ [S2, unofficial]. The exact function is not specified, but it adjusts each block by reconstruction accuracy.

**Parameters:**
*   $\alpha$: Momentum coefficient for EMA. Typical value: 0.999 [S1, S2, unofficial].
*   $\lambda$: Weight for distillation loss. Not given in the sources.
*   Block size/overlap: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** EMA updates can suffer from precision issues if not handled carefully, especially with mixed precision training [G].
*   **Boundaries:** Cross-block consistency is critical; poor boundary handling between blocks can lead to visible artifacts [S2, unofficial].
*   **Degenerate Input:** If a block has very low reconstruction accuracy, its weight might need to be capped to prevent destabilizing the teacher [G].

## Subtopics

### Distributed Learning for Large-Scale Scenes
A general approach to reconstructing large-scale scenes by dividing the problem into distributed parts [S3]. Momentum-GS builds on this by adding momentum self-distillation to ensure consistency across distributed blocks [S2, unofficial][S3].

## Operating characteristics
*   **Time Complexity:** Training time is reduced compared to CityGaussian due to fewer blocks required, but the EMA update adds a small overhead per step [S2, unofficial].
*   **Memory Complexity:** Requires storing both student and teacher parameters, doubling the memory for Gaussian parameters [S2, unofficial].
*   **Measured Performance:** Reports ~18.7% LPIPS improvement over CityGaussian with far fewer blocks [S2, unofficial].
*   **Strengths:** Decouples block count from GPU count, restores cross-block spatial consistency, improves convergence quality [S2, unofficial].
*   **Limitations:** Requires careful tuning of block weighting and distillation loss weight [S2, unofficial].
*   **When to Use:** Large-scale scene reconstruction where spatial consistency and convergence quality are critical [S2, unofficial].
*   **When Not to Use:** Small scenes where the overhead of distillation is not justified [G].

## Background
Momentum-GS was introduced as a 2026 SOTA update, marking a significant advancement in large-scene Gaussian Splatting training [S2, unofficial]. It addresses the challenges of distributed learning in large-scale scenes, such as geometry accuracy and reconstruction time bottlenecks [S3]. The method leverages momentum self-distillation, a technique previously used in other contexts, to guide the optimization of 3D Gaussian parameters [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
