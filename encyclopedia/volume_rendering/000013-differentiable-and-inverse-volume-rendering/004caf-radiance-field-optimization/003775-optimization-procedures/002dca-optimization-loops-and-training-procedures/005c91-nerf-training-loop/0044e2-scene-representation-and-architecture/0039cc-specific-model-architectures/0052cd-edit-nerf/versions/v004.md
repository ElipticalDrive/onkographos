# Edit-NeRF

Also called: EditNeRF

## Overview
Edit-NeRF is a method for user-controlled 3D scene editing that formulates the problem as a joint optimization of NeRF network parameters and latent codes [S1]. It solves the limitation of standard NeRFs, which are static and cannot be easily modified by external features, by introducing a modular network architecture with conditional radiance fields [S2]. It differs from its siblings by being the first method to execute user-controlled deformation models on NeRF using a shared shape network and instance-specific latent codes, whereas methods like CLIP-NeRF later introduced text/image guidance via CLIP embeddings [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** User-provided image conditioning (2D image or sketch) representing the desired edit [S1]; initial NeRF weights; category-specific shared shape network parameters.
*   **Output:** Edited 3D radiance field (density $\sigma$ and color $\mathbf{c}$) consistent with the user's edit.

**Data Structures:**
The shape representation is composed of two networks [S1]:
1.  **Shared Shape Network ($F_{share}$):** Category-specific, not conditioned on latent codes [S1]. It behaves as a deformation field [S1].
2.  **Instance-Specific Shape Network ($F_{inst}$):** Conditioned on a shape latent code $z_s$ [S1].

The color network is conditioned on an appearance latent code $z_a$ (or $z_c$ in some notations) [S1, S2].

**Numbered Steps:**
1.  **Initialization:** Train the model on multiple instances of the same category to learn underlying part features in an unsupervised manner [S3].
2.  **Latent Code Optimization:** Optimize the latent codes $z_s$ and $z_a$ to minimize the NeRF photometric loss against the user-provided image conditioning [S1].
3.  **Network Weight Optimization:** Optimize the MLP weights of the NeRF networks [S1].
4.  **Joint Optimization:** Optimize both the latent codes ($z_s, z_a$) and the MLP weights jointly to refine the edit [S1].

**Formulas:**
The rendered color $\hat{C}(\mathbf{r}, z_s, z_c)$ for a ray $\mathbf{r}$ is computed via volumetric rendering [S3]:
$$
\hat{C}(\mathbf{r}, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp \left( -\sum_{j=1}^{i-1} \sigma_j \delta_j \right)
$$
where:
*   $N_c$ is the number of samples along the ray [S3].
*   $c_i$ is the color at sample $i$ [G].
*   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the transmittance-derived alpha [S3].
*   $\sigma_j$ is the density at sample $j$ [S3].
*   $\delta_j$ is the distance between samples [S3].

**Parameters:**
*   $z_s$: Shape latent code.
*   $z_a$ (or $z_c$): Appearance/Color latent code.

**Edge Cases and Pitfalls:**
*   **Limited Shape Operations:** The method only supports adding or deleting local parts of an object; it does not support complex global deformations [S3].
*   **Slow Convergence:** The editing process is slow due to its iterative optimization nature [S3].

## Subtopics

### [Instruct-NeRF2NeRF](12265-instruct-nerf2nerf.md)
Instruct-NeRF2NeRF is a diffusion-based method for editing a NeRF scene guided by text instructions, utilizing InstructPix2Pix. It solves the problem of 3D scene modification by iteratively updating the training dataset with edited images and retraining the NeRF. It differs from its sibling [Edit-NeRF](9388-edit-nerf.md) by bypassing explicit 3D deformation fields or conditional radiance field architectures, instead relying on 2D image-level diffusion editing to drive 3D consistency.

### SINE
SINE is a method for editing a NeRF scene by editing a single view and applying the edit to the entire scene [S4]. It encodes changes in geometry and texture over the original NeRF scene by learning a prior-guided editing field [S4]. This field is used to render modified object geometry and color, and a color compositing layer supervised by the single edited view applies the edit on novel views [S4]. It is applied on real and synthetic scenes by changing geometry and texture of objects [S4].

## Operating characteristics
*   **Time Complexity:** Slow due to its iterative optimization nature [S3].
*   **Memory Complexity:** Requires storing latent codes $z_s$ and $z_a$ per instance/edit [S1].
*   **Strengths:** Allows user-controlled deformation; modular architecture separates shape and appearance [S2].
*   **Limitations:** Limited to local part addition/deletion; slow convergence [S3].
*   **When to Use:** When user-controlled local edits on a category of objects are required and computational time is not the primary constraint [S3].
*   **When Not to Use:** For global deformations, real-time editing, or when high geometric consistency is critical without additional constraints [S3].

## Background
Edit-NeRF was proposed by Liu et al. in 2021 [S3]. It was the first method to execute user-controlled deformation models on NeRF, proposing a modular network's conditional radiance field to guide model editing [S3]. It builds on the standard latent-conditioned NeRF architecture but introduces a shared shape network and instance-specific latent codes to enable editing [S1]. It was innovated upon by CLIP-NeRF in December 2021, which introduced text/image guidance via CLIP [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S3] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S4] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
