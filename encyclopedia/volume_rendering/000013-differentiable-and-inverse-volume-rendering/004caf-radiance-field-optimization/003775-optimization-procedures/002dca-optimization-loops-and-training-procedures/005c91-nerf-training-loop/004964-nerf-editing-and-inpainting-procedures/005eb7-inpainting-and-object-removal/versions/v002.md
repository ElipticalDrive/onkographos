# Inpainting and Object Removal

Also called: NeRF Inpainting, 3D Scene Inpainting, Object Removal in NeRFs

## Overview
This family of methods removes unwanted objects or fills occluded regions in a 3D scene represented by a pre-trained Neural Radiance Field (NeRF) [Parent]. Unlike standard NeRF training which reconstructs the entire scene from input views, these methods optimize the NeRF parameters to erase specific masked regions while maintaining geometric and photometric consistency across all unmasked views [Parent]. They are distinguished from sibling methods by their reliance on external 2D priors (such as inpainted images or diffusion models) to guide the 3D optimization, rather than relying solely on multi-view geometry or semantic segmentation alone [Parent].

## Shared principles
All methods in this family operate on a pre-trained NeRF representation $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, where $\mathbf{x}$ is a 3D coordinate, $\mathbf{d}$ is the viewing direction, $\sigma$ is volume density, and $\mathbf{c}$ is color [Parent]. The core mechanism involves an optimization loop that minimizes a loss function $L$ to update the NeRF parameters $\theta$ (or a subset thereof).

The general optimization objective is:
$$
L = L_{render} + \lambda_{guidance} L_{guidance} + \lambda_{reg} L_{reg}
$$
where $L_{render}$ is the standard photometric reconstruction loss on visible pixels, $L_{guidance}$ incorporates external priors (e.g., 2D inpainted images, diffusion model gradients, or semantic features), and $L_{reg}$ prevents artifacts or overfitting .

Key shared components include:
1.  **Masking/Selection:** A binary or probabilistic mask $M$ or selection field identifies the region of interest (ROI) or occluded areas. This can be user-defined, derived from semantic segmentation (e.g., LSeg, CLIP), or computed via geometric conflict detection .
2.  **External Priors:** Most methods rely on 2D priors to guide 3D changes. Common priors include:
    *   **2D Inpainting:** Using models like LaMa or Stable Diffusion to generate color/depth priors for occluded regions .
    *   **Diffusion Guidance:** Using Score Distillation Sampling (SDS) from text-to-image models (e.g., Stable Diffusion) to guide appearance and geometry .
    *   **Semantic/Text Features:** Using CLIP or LSeg to align rendered features with text prompts or semantic labels .
3.  **View Consistency Enforcement:** To prevent flickering or inconsistencies across views, methods enforce multi-view constraints. This is achieved through:
    *   **Geometric Consistency:** Using depth maps or epipolar constraints to ensure warped views align .
    *   **Gradient-Based Selection:** Adapting the reliance on 2D priors vs. multi-view geometry based on local texture gradients .
    *   **Reference Views:** Using a single or multiple inpainted reference views to propagate consistent content .

## Subtopics

### [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md)
A framework that removes objects using a single user-drawn mask and 2D color/depth guidance. It jointly minimizes color and depth losses to ensure consistency, avoiding the artifacts of using multiple inconsistent 2D priors.

### [Adaptive View Selection for Inpainting](12405-adaptive-view-selection-for-inpainting.md)
Addresses inconsistency by adaptively switching between 2D inpainting priors and multi-view geometry based on local texture gradients. It is chosen when maintaining multi-view consistency is critical to prevent flickering in textured regions.

### [Reference-guided Controllable Inpainting of NeRFs](13952-reference-guided-controllable-inpainting-of-nerfs.md)
Uses a single inpainted reference view to guide geometry and appearance via back-projection and a bilateral solver. It ensures view-dependent effects (VDEs) and avoids the quality deterioration seen in methods requiring multiple inpainted views.

## Background
The problem of 3D scene inpainting emerged as NeRFs became popular for novel view synthesis, where removing objects or filling holes required more than just 2D image editing. Early approaches adapted 2D inpainting techniques to 3D by optimizing NeRF parameters . The field has evolved to include diffusion-based guidance  and reference-guided consistency .

## Sources
[S1] NeRF-In: Neural Radiance Field Inpainting via Guided Depth Completion, https://arxiv.org/abs/2203.07112, 2022-03-14
[S2] Reference-Guided Controllable Inpainting of NeRFs, https://arxiv.org/abs/2303.13423, 2023-03-23
