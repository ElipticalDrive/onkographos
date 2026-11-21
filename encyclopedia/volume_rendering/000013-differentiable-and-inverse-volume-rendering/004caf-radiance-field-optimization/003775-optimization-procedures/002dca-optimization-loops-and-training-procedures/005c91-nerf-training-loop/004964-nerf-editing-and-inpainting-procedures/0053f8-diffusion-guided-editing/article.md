# Diffusion-guided editing
## Overview
Diffusion-guided editing methods modify pre-trained Neural Radiance Fields (NeRFs) using text prompts or image instructions by leveraging 2D text-to-image diffusion models as external priors [S1]. Unlike sibling methods that rely on semantic segmentation masks or geometric constraints for inpainting, this family uses diffusion gradients or iterative image synthesis to guide 3D changes [S1, 14365-repaint-nerf.md, 15089-instruct-nerf2nerf-optimization.md]. It is distinct from direct gradient-based guidance (like SDS) in variants that use iterative dataset updates, and from single-view guidance in variants that enforce multi-view consistency through iterative retraining [14365-repaint-nerf.md, 15089-instruct-nerf2nerf-optimization.md].
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

### [RePaint-NeRF](14365-repaint-nerf.md)
[RePaint-NeRF](14365-repaint-nerf.md) is a framework for editing 3D content in Neural Radiance Fields (NeRF) using text prompts by leveraging a pre-trained diffusion model to guide changes in designated 3D content, ensuring view consistency and scene integrity. It operates by iteratively alternating between two phases: (1) editing the 'dataset' of NeRF input images using an image-conditioned diffusion model, and (2) updating the underlying 3D representation (NeRF) to incorporate the edited images through standard NeRF training. This iterative process allows for the gradual percolation of diffusion priors into the 3D scene, resolving inconsistencies that arise from applying 2D edits independently to each view. It builds on [Diffusion-guided editing](15110-diffusion-guided-editing.md) by using an iterative dataset update scheme rather than direct gradient-based guidance (like SDS) for the NeRF parameters.

### [Instruct-NeRF2NeRF Optimization](15089-instruct-nerf2nerf-optimization.md)
[Instruct-NeRF2NeRF Optimization](15089-instruct-nerf2nerf-optimization.md) is a method for editing pre-trained Neural Radiance Fields (NeRFs) using natural language instructions. It solves the problem of modifying 3D scene appearance and geometry without retraining from scratch by leveraging a 2D text-to-image diffusion model as a guidance signal. Unlike sibling methods that may use semantic segmentation or geometric constraints for inpainting, this approach relies on InstructPix2Pix to generate edited views, which are then backpropagated to update the NeRF parameters. It is distinct in its use of classifier-free guidance weights to control edit strength and its lack of a single convergence point due to the stochastic nature of the diffusion model.

### MALD-NeRF
MALD-NeRF employs a per-scene customization latent diffusion model to generate inpainted images, iteratively updating the training set [S1]. It incorporates a masked patch-based adversarial training strategy with adversarial loss and discriminator feature matching losses to refine results [S1]. The loss function includes a pixel-level regression loss for reconstruction [S1].

## Background
Diffusion-guided editing emerged as a solution to the limitations of single-view editing and the inconsistency of applying 2D edits to 3D scenes independently. Early methods relied on Score Distillation Sampling (SDS) for direct gradient guidance, but iterative dataset update schemes like RePaint-NeRF were introduced to better preserve view consistency and scene integrity [14365-repaint-nerf.md]. The field has increasingly focused on exploring inpainting techniques on 3DGS, benefiting from its outstanding rendering efficiency and high-quality scene reconstruction [S1].
## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
