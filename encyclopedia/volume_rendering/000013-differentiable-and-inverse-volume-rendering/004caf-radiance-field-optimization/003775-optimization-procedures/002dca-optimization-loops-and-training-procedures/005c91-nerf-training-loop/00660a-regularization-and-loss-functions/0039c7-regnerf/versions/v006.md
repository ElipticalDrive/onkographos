# RegNeRF

Also called: Regularized NeRF

## Overview
RegNeRF is a method for synthesizing novel views from sparse input images by applying depth smoothness and color priors as regularization terms during training [S1]. It solves the problem of NeRF overfitting and unrealistic interpolation in few-shot scenarios without requiring scene-specific pretraining [S1]. Unlike sibling methods that rely on pretrained feature extractors or adversarial training, RegNeRF constrains unobserved views using a Normalizing Flow prior for color and a geometric smoothness prior for depth, achieving performance comparable to pretrained models on sparse datasets [S1].

## How it works
RegNeRF augments the standard NeRF training loop with two specific regularization losses and a sampling strategy to stabilize early training [S1]. It uses Mip-NeRF as the backbone representation [S1].

**Inputs and Outputs:**
*   **Inputs:** A sparse set of $N$ input images with known poses. Randomly sampled unobserved viewpoints are generated during training to compute regularization losses.
*   **Outputs:** A trained neural radiance field (MLP parameters) that renders photorealistic images from novel viewpoints.

**Data Structures:**
*   **NeRF Backbone:** Mip-NeRF representation, encoding position $\mathbf{x}$ and viewing direction $\mathbf{v}$ into density $\sigma$ and color $\mathbf{c}$ [S1].
*   **Normalizing Flow Model:** A pre-trained bijection $\phi$ mapping image patches to a latent space, trained on a large unposed dataset (e.g., JFT-300M) [S1]. The latent distribution $p_z$ is Gaussian [S1].
*   **Patches:** Small spatial regions of size $S_{patch} \times S_{patch}$ extracted from rendered views [S1].

**Parameters:**
*   $S_{patch}$: Patch size for regularization (Not given in the sources).
*   $\lambda_{ds}$: Weight for depth smoothness loss (Not given in the sources).
*   $\lambda_{nll}$: Weight for color negative log-likelihood loss (Not given in the sources).
*   $V_{small}$: Initial small volume for sampled space annealing (Not given in the sources).

**Numbered Steps:**

1.  **Sampled Space Annealing (Initialization):** At the start of training, limit the range of sample points along rays to a small volume $V_{small}$ defined for all input images. This prevents divergent NeRF modes with high density at the ray origin. Gradually extend the sampling range to the full scene volume as training progresses [S1].
2.  **Render Patches:** For each training step, render color patches $\mathbf{P}_r$ and expected depth maps $d$ from both input views and randomly sampled unobserved viewpoints [S1].
3.  **Compute Depth Smoothness Loss ($\mathcal{L}_{ds}$):** Calculate the smoothness of the expected depth field on patches from unobserved viewpoints. For a patch of size $S_{patch}$, let $d(r^i_j)$ be the expected depth of the ray through pixel $(i,j)$ [S1]:
    $$ \mathcal{L}_{ds} = \sum_{i,j} \left[ (d(r^i_j) - d(r^{i+1}_j))^2 + (d(r^i_j) - d(r^i_{j+1}))^2 \right] $$
    This encourages the world geometry to be piecewise smooth [S1].
4.  **Compute Color Regularization Loss ($\mathcal{L}_{NLL}$):** Estimate the likelihood of rendered color patches using the pre-trained Normalizing Flow model. Let $\phi$ be the learned bijection from patches to $\mathbb{R}^d$ where $d = S_{patch} \cdot S_{patch} \cdot 3$, and $p_z$ be the Gaussian latent distribution [S1]:
    $$ \mathcal{L}_{NLL} = - \sum_{r} \log p_z(\phi(\mathbf{P}_r)) $$
    Maximizing this likelihood constrains the color distribution to match natural image statistics [S1].
5.  **Optimize:** Minimize the total loss, combining the standard photometric reconstruction loss (MSE) with the regularization terms:
    $$ \mathcal{L}_{total} = \mathcal{L}_{photo} + \lambda_{ds} \mathcal{L}_{ds} + \lambda_{NLL} \mathcal{L}_{NLL} $$
    Backpropagate gradients through the differentiable volume rendering process to update the NeRF parameters [S1].

**Edge Cases and Pitfalls:**
*   **Ray Origin Density:** Without sampled space annealing, the NeRF may converge to a mode with artificially high density at the camera origin, causing artifacts [S1].
*   **Unobserved View Sampling:** The regularization relies on randomly sampled unobserved viewpoints. If the sampling distribution is poor, the priors may not generalize effectively [S1].

## Subtopics

### DiffNeRF
DiffNeRF is a regularization framework that uses differential geometry to improve robustness with sparse or inconsistent data [S1, 8572-diffnerf.md]. Unlike RegNeRF, which approximates gradients via finite differences on patch-based dataloaders, DiffNeRF uses automatic differentiation to compute depth gradients directly from the ray-based NeRF representation, allowing regularization of unseen views without specialized dataloader modifications [S1, 8572-diffnerf.md]. Choose DiffNeRF when automatic differentiation is preferred over finite difference approximations for depth smoothness.

### SS-NeRF
SS-NeRF employs an encoding function and two position decoding functions (one direction-dependent, one direction-independent) represented by MLPs to produce various scene properties [S1, 10636-specific-loss-formulations.md]. It is trained to produce color, semantic labels, surface normal, shading, keypoints, and edges using a combination of losses: MSE for color and surface normals; MAE for shading, keypoints and edges; and cross-entropy for semantic labels [S1, 10636-specific-loss-formulations.md]. Choose SS-NeRF when multi-task scene property synthesis is required alongside view synthesis.

## Operating characteristics
*   **Time and Memory Complexity:** RegNeRF requires rendering additional patches from randomly sampled unobserved viewpoints, increasing computational cost per iteration compared to standard NeRF [S1]. The Normalizing Flow inference adds overhead for color regularization [S1].
*   **Measured Performance:** Tested on DTU and LLFF datasets, RegNeRF outperformed PixelNeRF, SRF, and MVSNeRF under sparse view conditions [S1]. It achieved performance comparable to models pretrained on DTU and fine-tuned per scene, while outperforming Mip-NeRF and DietNeRF [S1].
*   **Strengths:** Does not require pretraining on the target dataset; effective for sparse input views; improves geometric smoothness and color realism [S1].
*   **Limitations:** Relies on the quality of the pre-trained Normalizing Flow model; performance depends on the random sampling of unobserved views [S1].
*   **When to use:** Sparse view synthesis where pretraining is not feasible or undesirable.
*   **When not to use:** Dense view settings where standard NeRF or Mip-NeRF suffices; scenes with highly discontinuous geometry where smoothness priors may blur details.

## Background
RegNeRF was proposed in December 2021 (published in CVPR 2022) by K. Gao et al. [S1]. It was developed to address the limitations of NeRF in sparse view scenarios, where standard training leads to overfitting and poor interpolation [S1]. The method builds on Mip-NeRF [48] as the backbone representation [S1]. It was tested on the DTU [29] and LLFF [6] datasets [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
