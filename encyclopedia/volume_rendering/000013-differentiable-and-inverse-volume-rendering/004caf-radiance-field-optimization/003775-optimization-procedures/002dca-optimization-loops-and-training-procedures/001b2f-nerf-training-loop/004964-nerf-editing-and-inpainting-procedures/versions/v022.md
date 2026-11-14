# NeRF Editing and Inpainting Procedures

Also called: NeRF Editing, NeRF Inpainting, 3D Scene Editing

## Overview
NeRF Editing and Inpainting Procedures modify a pre-trained Neural Radiance Field (NeRF) to remove objects, change appearance, or fill occluded regions while maintaining multi-view consistency. Unlike standard training which optimizes for reconstruction of input views, these methods optimize for a target state defined by external priors (e.g., 2D inpainting, diffusion models, or semantic masks). They are distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by their use of auxiliary guidance signals beyond photometric error, and from [NeRF Training Loop](970-nerf-training-loop.md) by their focus on localized or semantic modifications rather than global scene reconstruction.

## Shared principles
All methods in this family operate on a pre-trained NeRF representation $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, where $\mathbf{x}$ is a 3D coordinate, $\mathbf{d}$ is the viewing direction, $\sigma$ is volume density, and $\mathbf{c}$ is color [Parent]. The core mechanism involves an optimization loop that minimizes a loss function $L$ to update the NeRF parameters $\theta$ (or a subset thereof).

The general optimization objective is:
$$
L = L_{render} + \lambda_{guidance} L_{guidance} + \lambda_{reg} L_{reg}
$$
where $L_{render}$ is the standard photometric reconstruction loss on visible pixels, $L_{guidance}$ incorporates external priors (e.g., 2D inpainted images, diffusion model gradients, or semantic features), and $L_{reg}$ prevents artifacts or overfitting [S1, 12372-nerf-in-inpainting-optimization.md, 13438-nerf-art.md].

Key shared components include:
1.  **Masking/Selection:** A binary or probabilistic mask $M$ or selection field identifies the region of interest (ROI) or occluded areas. This can be user-defined, derived from semantic segmentation (e.g., LSeg, CLIP), or computed via geometric conflict detection [S1, 14365-repaint-nerf.md, 8658-progressive-inpainting-and-erasing.md].
2.  **External Priors:** Most methods rely on 2D priors to guide 3D changes. Common priors include:
    *   **2D Inpainting:** Using models like LaMa or Stable Diffusion to generate color/depth priors for occluded regions [S1, 12405-adaptive-view-selection-for-inpainting.md].
    *   **Diffusion Guidance:** Using Score Distillation Sampling (SDS) from text-to-image models (e.g., Stable Diffusion) to guide appearance and geometry [S1, 14365-repaint-nerf.md].
    *   **Semantic/Text Features:** Using CLIP or LSeg to align rendered features with text prompts or semantic labels [S1, 13438-nerf-art.md, 14365-repaint-nerf.md].
3.  **View Consistency Enforcement:** To prevent flickering or inconsistencies across views, methods enforce multi-view constraints. This is achieved through:
    *   **Geometric Consistency:** Using depth maps or epipolar constraints to ensure warped views align [S1, 11860-deceptive-nerf-3dgs-pipeline.md].
    *   **Gradient-Based Selection:** Adapting the reliance on 2D priors vs. multi-view geometry based on local texture gradients [S1, 12405-adaptive-view-selection-for-inpainting.md].
    *   **Reference Views:** Using a single or multiple inpainted reference views to propagate consistent content [S1, 13952-reference-guided-controllable-inpainting-of-nerfs.md].

## Subtopics

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Reconstructs a panoramic NeRF from a single input by iteratively expanding viewpoint coverage, solving geometry conflicts between visible and invisible regions.

### [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)
Enhances sparse-view reconstruction by generating pseudo-observations via a diffusion model conditioned on rendered images and uncertainty maps.

### [Blended-NeRF](12273-blended-nerf.md)
Edits specific regions of interest (ROI) using text prompts and a 3D bounding box, blending synthesized content with the original scene via volumetric blending.

### [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md)
Removes unwanted objects by optimizing model parameters using user masks and external RGB-D priors, jointly optimizing color and depth guidance.

### [Adaptive View Selection for Inpainting](12405-adaptive-view-selection-for-inpainting.md)
Addresses inconsistency artifacts by adaptively switching between 2D inpainted color priors and multi-view geometric consistency based on regional average gradients.

### [NeRF Editing Procedure](12491-nerf-editing-procedure.md)
Enables direct 3D editing by decoupling semantic selection from appearance rendering using a Distilled Feature Field (DFF), avoiding iterative gradient descent for the edit itself.

### [NeRF-Art](13438-nerf-art.md)
Stylizes pre-trained NeRFs using text prompts, altering both appearance and 3D geometry via global-local contrastive learning and directional constraints.

### [Reference-guided Controllable Inpainting of NeRFs](13952-reference-guided-controllable-inpainting-of-nerfs.md)
Performs 3D inpainting using a single inpainted reference view to guide geometry and appearance, ensuring view consistency and generating view-dependent effects.

### [RePaint-NeRF Limitations](14340-repaint-nerf-limitations.md)
Documents empirical limitations of RePaint-NeRF, including computational efficiency, geometric fidelity under shape changes, and view-angle consistency.

### [RePaint-NeRF](14365-repaint-nerf.md)
Edits 3D content using text prompts and diffusion models, extracting view-consistent semantic masks to isolate target objects and guide changes.

### [Semantic and Language-Guided Editing](14393-semantic-and-language-guided-editing.md)
Modifies NeRFs using high-level semantic cues (text, language, semantic masks) integrated into the optimization loop to guide 3D structure and appearance.

### [InNeRF360](14845-innerf360.md)
Combines multiview-consistent 2D priors with 3D diffusion guidance, decoupling plausible content generation (2D inpainting) from 3D reconstruction refinement.
*   #### Post-Refinement Strategy
    *   **Definition:** A strategy employed after the NeRF has been trained on updated data to enhance consistency by generating a more accurate target for each view using the modified Instruct-Pix2Pix with averaging architecture.
    *   **Step:** Use the rendered image as input while continuing to use the mixup image as a condition.
*   #### MALD-NeRF
    *   **Definition:** A method employing a per-scene customization latent diffusion model to generate inpainted images, iteratively updating the training set.
    *   **Step:** Incorporates a masked patch-based adversarial training strategy with adversarial loss and discriminator feature matching losses to refine results.
    *   **Parameter:** Loss function: Pixel-level regression loss for reconstruction.

## Background
NeRF editing and inpainting emerged as a natural extension of NeRF optimization, shifting focus from reconstruction to manipulation. Early methods relied on direct optimization of NeRF parameters with auxiliary losses [12372-nerf-in-inpainting-optimization.md]. The integration of 2D generative models, particularly Stable Diffusion, enabled text-guided editing [14365-repaint-nerf.md, 13438-nerf-art.md]. Recent approaches emphasize view consistency through geometric constraints [8658-progressive-inpainting-and-erasing.md, 11860-deceptive-nerf-3dgs-pipeline.md] and semantic priors [14393-semantic-and-language-guided-editing.md].

## Sources
[S1] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
