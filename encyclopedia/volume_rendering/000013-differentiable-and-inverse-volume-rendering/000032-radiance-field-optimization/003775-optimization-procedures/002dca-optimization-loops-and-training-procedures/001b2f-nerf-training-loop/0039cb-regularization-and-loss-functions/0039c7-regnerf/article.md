# RegNeRF

Also called: Regularized NeRF

## Overview
RegNeRF is a regularization framework for Neural Radiance Fields (NeRF) designed to improve view synthesis quality from sparse input views by constraining unobserved regions [S1, unofficial]. It distinguishes itself from siblings like [DiffNeRF](8572-diffnerf.md) by approximating spatial gradients via finite differences on patch-based dataloaders, whereas DiffNeRF uses automatic differentiation on ray-based representations [S1, unofficial].

## How it works
RegNeRF augments the standard photometric loss with two specific regularization terms to prevent overfitting and ensure geometric plausibility in sparse scenarios [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** Sparse set of input images $\{I_i\}$, corresponding camera poses, and a NeRF model parameterized by $\theta$.
*   **Outputs:** Optimized NeRF parameters $\theta$ that render high-quality novel views.

**Data Structures:**
*   **Patch-based Dataloader:** Unlike standard ray-sampling, RegNeRF samples spatial patches from input images to compute local smoothness constraints [S1, unofficial].
*   **Normalizing Flow Prior:** A generative model $p(\mathbf{x})$ used to constrain the latent space of the NeRF outputs [S1, unofficial].

**Loss Function:**
The total loss is defined as:
$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \lambda_{\text{smooth}} \mathcal{L}_{\text{smooth}} + \lambda_{\text{prior}} \mathcal{L}_{\text{prior}} $$
where $\mathcal{L}_{\text{photo}}$ is the standard MSE reconstruction loss [S1, unofficial].

1.  **Smoothness Prior ($\mathcal{L}_{\text{smooth}}$):** Enforces spatial smoothness on the density and color fields. It is computed by sampling adjacent pixels in the input image space and penalizing large differences in the predicted density $\sigma$ and color $\mathbf{c}$ [S1, unofficial].
    $$ \mathcal{L}_{\text{smooth}} = \sum_{i,j} \left( ||\nabla \sigma_i - \nabla \sigma_j||_2^2 + ||\nabla \mathbf{c}_i - \nabla \mathbf{c}_j||_2^2 \right) $$
    Gradients are approximated using finite differences on the patch-based samples [S1, unofficial].

2.  **Normalizing Flow Prior ($\mathcal{L}_{\text{prior}}$):** Constrains the distribution of the NeRF's output features to match a learned prior distribution $p(\mathbf{z})$ using a normalizing flow model [S1, unofficial].
    $$ \mathcal{L}_{\text{prior}} = -\log p_{\text{flow}}(\mathbf{z}) $$
    where $\mathbf{z}$ represents the latent features extracted from the NeRF [S1, unofficial].

**Parameters:**
*   $\lambda_{\text{smooth}}$: Weight for the smoothness term. Not given in the sources.
*   $\lambda_{\text{prior}}$: Weight for the normalizing flow prior. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sparse Input:** The method is specifically designed for sparse views; performance on dense inputs may not offer significant gains over standard NeRF [S1, unofficial].
*   **Finite Difference Approximation:** Using finite differences for smoothness may introduce discretization artifacts if patch sizes are not chosen carefully [S1, unofficial].

## Subtopics

### [DiffNeRF](8572-diffnerf.md)
Uses automatic differentiation for depth gradients directly from ray-based NeRF, avoiding patch-based dataloader modifications. Choose when automatic differentiation is preferred over finite differences.

## Operating characteristics
*   **Time Complexity:** Training time increases due to the additional computation of smoothness priors via finite differences and the evaluation of the normalizing flow prior [S1, unofficial].
*   **Memory Complexity:** Requires storing the normalizing flow model and patch-based sampling buffers [S1, unofficial].
*   **Strengths:** Effective for sparse view synthesis; prevents overfitting and unrealistic interpolation [S1, unofficial].
*   **Limitations:** Relies on patch-based sampling which may not capture global geometry as well as ray-based methods; requires careful tuning of regularization weights [S1, unofficial].
*   **Performance:** Evaluated on the DTU dataset, showing improved results over standard NeRF in sparse settings [S1, unofficial].

## Background
RegNeRF was introduced in "Regularizing Neural Radiance Fields for View Synthesis from Sparse Inputs" at CVPR 2022 [S1, unofficial]. It addresses the problem of MLP overfitting and poor generalization in few-shot scenarios by incorporating generative and smoothness priors [S1, unofficial].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
