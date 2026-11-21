# Nerfiller

Also called: Grid-based Diffusion Inpainting for NeRF

## Overview
Nerfiller is a method for 3D scene completion using arbitrary masks, leveraging a grid-based prior and an iterative distillation framework [S1]. It differs from sibling methods like [Reference-guided Controllable Inpainting of Neural Radiance Fields](15289-reference-guided-controllable-inpainting-of-neural-radiance-.md) and [Masked Completion via GAN Inversion](20185-masked-completion-via-gan-inversion.md) by explicitly shuffling rendered images into 2x2 grids to feed into 2D inpainting diffusion models, thereby enforcing 3D consistency through multi-view scoring [S1]. While [Masked Completion via GAN Inversion](20185-masked-completion-via-gan-inversion.md) relies on GAN inversion for completion, Nerfiller uses diffusion priors to generate harmonious completions for large, arbitrary masked regions [S1].

## How it works
**Inputs:**
*   A pre-trained 2D inpainting diffusion model.
*   A text description of the scene content to be completed [S1].
*   A 3D mask defining the region to be completed [S1].
*   An initial NeRF representation (or empty scene) [S1].

**Outputs:**
*   A completed 3D radiance field (NeRF) consistent with the text description and unmasked regions [S1].

**Data Structures:**
*   **NeRF Grid:** Standard multi-resolution hash grid or voxel grid storing density and color features [S1].
*   **Image Grid:** A temporary structure where $N$ rendered images are shuffled and arranged into $2 \times 2$ composite grids [S1].

**Parameters:**
*   **Text Prompt:** Describes the desired content for the masked region [S1].
*   **Diffusion Model:** A 2D inpainting model capable of conditional generation [S1].
*   **Multi-view Scoring Weight:** Balances consistency across different views [S1].

**Steps:**
1.  **Initialization:** Initialize the NeRF parameters. [G] Typically random initialization or from a coarse reconstruction.
2.  **Rendering:** Render a set of views of the current NeRF from various camera poses.
3.  **Grid Construction:** Shuffle the rendered images and arrange them into $2 \times 2$ grids. This shuffling helps break view-specific biases and encourages the diffusion model to learn 3D-consistent features rather than 2D artifacts [S1].
4.  **Diffusion Inpainting:** Feed the $2 \times 2$ grids into the 2D inpainting diffusion model, conditioned on the text prompt and the mask. The model generates inpainted images for the masked regions.
5.  **Distillation/Update:** Distill the 2D inpainted results back into the 3D NeRF representation. This is done by minimizing a loss function that aligns the rendered views of the NeRF with the inpainted images from the diffusion model.
6.  **Multi-view Scoring:** Apply a multi-view scoring mechanism to evaluate the consistency and sharpness of the generated content across different views, particularly under large view variations [S1]. This score guides the optimization to ensure 3D coherence.
7.  **Iterative Refinement:** Repeat steps 2-6 iteratively. The framework distills the 2D inpainting results into a single 3D scene over multiple iterations [S1].

**Formulas:**
*   The specific loss function for distillation is not explicitly detailed in the provided notes, but it leverages Score Distillation Sampling (SDS) to align rendered views with diffusion priors [S1].
*   Let $I_{rendered}$ be the rendered image, $I_{inpainted}$ be the diffusion-generated image, and $L$ be the distillation loss. The optimization minimizes $L(I_{rendered}, I_{inpainted})$.

**Edge Cases and Pitfalls:**
*   **Large View Variations:** The method is designed to handle large view variations, but consistency may still be challenging if the text prompt is ambiguous [S1].
*   **Arbitrary Masks:** The method supports arbitrary masks, but complex mask shapes may require more iterations for convergence [S1].

## Subtopics

### Masked Radiance Field Completion
A technique to synthesize a completion of a masked region in a radiance field that harmonizes with the non-masked region, given an input radiance field and a 3D mask [S2]. It is distinct from Nerfiller's grid-based approach by using a conditional diffusion sampling process directly on the voxel grid [S2]. Choose this when a direct voxel-level completion is preferred over iterative distillation from 2D models.

**Steps:**
1.  **Conditional Guidance:** Perform conditional completion by gradually guiding the unconditional sampling process in the known region to the input radiance field $f_{in}$ using the update rule:
    $$ f_{t-1} = \bar{\alpha}_t (m \tilde{f}_t + (1 - m) f_{in}) $$
    where $m$ is a binary mask applied to the input voxel grid, $\tilde{f}_t$ is the denoised estimate at step $t$, and $\bar{\alpha}_t$ is the cumulative noise schedule coefficient [S2].
2.  **Sampling:** Sample the noisy latent at step $t-1$ from the distribution:
    $$ f_{t-1} \sim \mathcal{N}(f_0, (1 - \bar{\alpha}_t)I) $$
    where $f_0$ is the initial noise [S2].
3.  **Fusion:** Iteratively fuse the non-masked input region with the estimated denoising $\tilde{f}_0^t$ and apply the diffusion process on the resulting $f_0^{t-1}$ to obtain the completion [S2].

**Parameters:**
*   $m$: A binary mask applied to the input voxel grid, indicating known (1) and masked (0) regions [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Iterative nature implies multiple forward passes through the diffusion model and NeRF renderer.
*   **Memory Complexity:** Not given in the sources. Depends on the NeRF grid size and the batch size of rendered images.
*   **Strengths:** Handles arbitrary masks effectively; leverages strong 2D diffusion priors for high-quality texture; multi-view scoring improves consistency [S1].
*   **Limitations:** Not given in the sources for Nerfiller specifically, though related diffusion methods note limitations compared to GAN-based approaches in terms of diversity or fidelity [S2].
*   **When to use:** When scene completion with arbitrary masks is required, and high-quality texture synthesis is prioritized over geometric precision [S1].
*   **When not to use:** When real-time performance is critical, or when the mask is very small and tight (object removal), where other methods might be more efficient [S1].

## Background
Nerfiller was introduced as a method for 3D scene completion with arbitrary masks, building on the capabilities of 2D inpainting diffusion models [S1]. It was developed to address the limitations of previous methods that struggled with large, arbitrary masks [S1]. The method is part of the broader trend of using diffusion models for 3D editing and completion [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] DiffRF: Rendering-Guided 3D Radiance Field Diffusion, https://doi.org/10.1109/cvpr52729.2023.00421, fetched 2026-10-26
