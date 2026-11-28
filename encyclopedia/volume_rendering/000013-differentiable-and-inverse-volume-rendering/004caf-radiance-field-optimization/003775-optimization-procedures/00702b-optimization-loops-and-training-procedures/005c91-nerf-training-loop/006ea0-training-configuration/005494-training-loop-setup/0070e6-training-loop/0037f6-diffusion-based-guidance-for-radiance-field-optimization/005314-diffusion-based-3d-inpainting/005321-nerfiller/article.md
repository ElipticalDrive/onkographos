# Nerfiller


## Overview
Nerfiller is a method for 3D scene completion using arbitrary masks, leveraging a grid-based prior and an iterative distillation framework [S1]. It differs from sibling methods like [Reference-guided Controllable Inpainting of Neural Radiance Fields](15289-reference-guided-controllable-inpainting-of-neural-radiance-.md) and [Masked Completion via GAN Inversion](20185-masked-completion-via-gan-inversion.md) by explicitly shuffling rendered images into 2x2 grids to feed into 2D inpainting diffusion models, thereby enforcing 3D consistency through multi-view scoring [S1]. While [Masked Completion via GAN Inversion](20185-masked-completion-via-gan-inversion.md) relies on GAN inversion for completion, Nerfiller uses diffusion priors to generate harmonious completions for large, arbitrary masked regions [S1].

## How it works
**Inputs:**
*   A pre-trained 2D inpainting diffusion model [S1].
*   A text description of the scene content to be completed [S1].
*   A 3D mask defining the region to be completed [S1].
*   An initial NeRF representation [G].

**Outputs:**
*   A completed 3D radiance field consistent with the text description and unmasked regions [S1].

**Data Structures:**
*   **Image Grid:** A temporary structure where rendered images are shuffled and arranged into $2 \times 2$ composite grids [S1].

**Parameters:**
*   **Text Prompt:** Describes the desired content for the masked region [S1].
*   **Diffusion Model:** A 2D inpainting model capable of conditional generation [S1].

**Steps:**
1.  **Rendering:** Render a set of views of the current NeRF from various camera poses [G].
2.  **Grid Construction:** Shuffle the rendered images and arrange them into $2 \times 2$ grids [S1].
3.  **Diffusion Inpainting:** Feed the $2 \times 2$ grids into the 2D inpainting diffusion model, conditioned on the text prompt and the mask [S1]. The model generates inpainted images for the masked regions [S1].
4.  **Distillation/Update:** Distill the 2D inpainted results back into the 3D NeRF representation [S1]. This is done by minimizing a loss function that aligns the rendered views of the NeRF with the inpainted images from the diffusion model [S1].
5.  **Multi-view Scoring:** Apply a multi-view scoring mechanism to evaluate the consistency and sharpness of the generated content across different views, particularly under large view variations [S1]. This score guides the optimization to ensure 3D coherence [S1].
6.  **Iterative Refinement:** Repeat steps 1-5 iteratively to distill the 2D inpainting results into a single 3D scene [S1].

**Formulas:**
*   The method leverages Score Distillation Sampling (SDS) to align rendered views with diffusion priors based on a text description [S1].
*   The specific mathematical formulation of the distillation loss is not detailed in the sources.

**Edge Cases and Pitfalls:**
*   **Large View Variations:** The method incorporates a multi-view scoring mechanism to improve consistency and sharpness, especially under large view variations [S1].
*   **Arbitrary Masks:** The method focuses on scene completion with arbitrary masks rather than object removal with tight masks [S1].

## Subtopics

### Masked Radiance Field Completion
A technique to synthesize a completion of a masked region in a radiance field that harmonizes with the non-masked region, given an input radiance field and a 3D mask [S2]. It is distinct from Nerfiller's grid-based approach by using a conditional diffusion sampling process directly on the voxel grid [S2].

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
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Handles arbitrary masks effectively; leverages strong 2D diffusion priors for high-quality texture; multi-view scoring improves consistency [S1].
*   **Limitations:** Not given in the sources for Nerfiller specifically.
*   **When to use:** When scene completion with arbitrary masks is required [S1].
*   **When not to use:** Not given in the sources.

## Background
Nerfiller was introduced as a method for 3D scene completion with arbitrary masks, building on the capabilities of 2D inpainting diffusion models [S1]. It was developed to address the limitations of previous methods that struggled with large, arbitrary masks [S1]. The method is part of the broader trend of using diffusion models for 3D editing and completion [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] DiffRF: Rendering-Guided 3D Radiance Field Diffusion, https://doi.org/10.1109/cvpr52729.2023.00421, fetched 2026-10-26
