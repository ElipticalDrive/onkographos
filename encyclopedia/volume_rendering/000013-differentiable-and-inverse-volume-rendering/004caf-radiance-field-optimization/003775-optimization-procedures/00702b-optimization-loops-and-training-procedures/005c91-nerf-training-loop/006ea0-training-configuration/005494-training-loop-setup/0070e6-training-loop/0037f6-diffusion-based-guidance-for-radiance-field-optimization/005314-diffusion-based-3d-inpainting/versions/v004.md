# Diffusion-Based 3D Inpainting

Also called: Inpaint3D

## Overview
Diffusion-Based 3D Inpainting is a method that utilizes a self-trained 2D inpainting diffusion model to complete 3D scenes or radiance fields where specific regions are masked out [S1]. It solves the problem of generating geometrically and photometrically consistent content for missing areas by combining a reconstruction loss for observed regions with a Score Distillation Sampling (SDS) loss for masked regions [S1]. It differs from sibling methods like [Iterative Optimization with Learnable Parameters for Diffusion-Guided Radiance Fields](8945-iterative-optimization-with-learnable-parameters-for-diffusi.md) by explicitly separating the supervision signal into unmasked reconstruction and masked diffusion priors, rather than relying solely on general score distillation [S1].

## How it works
**Inputs:**
*   A 3D representation (e.g., NeRF or 3D Gaussian Splatting) initialized from input views.
*   A set of input images with corresponding binary masks indicating regions to be inpainted.
*   A pre-trained 2D inpainting diffusion model (e.g., Latent Diffusion Model adapted for inpainting) [S1].

**Outputs:**
*   An optimized 3D representation with the masked regions filled in.

**Data Structures:**
*   **3D Representation:** The specific layout depends on the base renderer (e.g., voxel grid for NeRF, point cloud with attributes for 3DGS). Not given in the sources.
*   **Diffusion Model:** A standard U-Net architecture with attention layers, conditioned on text prompts and masks. Not given in the sources.

**Parameters:**
*   **Reconstruction Loss Weight:** Balances the fidelity of unmasked regions. Not given in the sources.
*   **SDS Loss Weight:** Balances the adherence to the diffusion prior in masked regions. Not given in the sources.
*   **Diffusion Timesteps:** The noise levels $t$ used during the sampling process for SDS computation. Not given in the sources.

**Steps:**
1.  **Render Views:** Render the current 3D representation from the input camera viewpoints to obtain predicted RGB images $I_{pred}$.
2.  **Compute Reconstruction Loss:** For pixels where the mask $M$ is 0 (unmasked), compute the pixel-wise difference between the predicted image $I_{pred}$ and the ground truth input image $I_{gt}$.
    $$ L_{rec} = \sum_{p \in \Omega_{unmasked}} \| I_{pred}(p) - I_{gt}(p) \|^2 $$
    where $\Omega_{unmasked}$ is the set of unmasked pixel coordinates [S1].
3.  **Compute SDS Loss:** For pixels where the mask $M$ is 1 (masked), compute the Score Distillation Sampling (SDS) loss. This involves:
    a. Adding noise $\epsilon \sim \mathcal{N}(0,1)$ to the predicted image $I_{pred}$ at a random timestep $t$ to get $I_t$.
    b. Feeding $I_t$ and the mask $M$ into the frozen 2D inpainting diffusion model to predict the noise $\hat{\epsilon}_\theta(I_t, M, t)$.
    c. Calculating the gradient of the predicted noise with respect to the input image $I_{pred}$, scaled by the noise schedule derivative $\frac{\partial x_t}{\partial \epsilon}$ [S1].
    $$ L_{SDS} = \mathbb{E}_{t, \epsilon} \left[ w(t) \cdot \mathbb{E}_{\epsilon} \left[ \left( \hat{\epsilon}_\theta(I_t, M, t) - \epsilon \right) \frac{\partial I_{pred}}{\partial I_{pred}} \right] \right] $$
    The gradient is backpropagated through the differentiable renderer to update the 3D representation parameters [S1].
4.  **Optimize:** Minimize the total loss $L_{total} = L_{rec} + \lambda L_{SDS}$ with respect to the 3D representation parameters, where $\lambda$ is a hyperparameter balancing the two losses [S1].
5.  **Iterate:** Repeat steps 1-4 for a fixed number of optimization steps or until convergence.

**Pitfalls:**
*   **Inconsistency:** Without multi-view scoring or consistency constraints, the inpainted regions may vary across different viewpoints. [S1] notes that methods like MVIPNeRF incorporate multi-view scoring to improve consistency.
*   **Artifact Propagation:** Errors in the initial 3D reconstruction can propagate into the inpainted regions. [S1] mentions that some methods use depth-warping refinement to obtain view-consistent 2D segmentations before training.

## Subtopics

### [Nerfiller](14835-nerfiller.md)
Nerfiller is a method for 3D scene completion using arbitrary masks, leveraging a grid-based prior and an iterative distillation framework. It differs from sibling methods like [Reference-guided Controllable Inpainting of Neural Radiance Fields](15289-reference-guided-controllable-inpainting-of-neural-radiance-.md) and [Masked Completion via GAN Inversion](20185-masked-completion-via-gan-inversion.md) by explicitly shuffling rendered images into 2x2 grids to feed into 2D inpainting diffusion models, thereby enforcing 3D consistency through multi-view scoring. While [Masked Completion via GAN Inversion](20185-masked-completion-via-gan-inversion.md) relies on GAN inversion for completion, Nerfiller uses diffusion priors to generate harmonious completions for large, arbitrary masked regions.

### [Reference-guided Controllable Inpainting of Neural Radiance Fields](15289-reference-guided-controllable-inpainting-of-neural-radiance-.md)
This method performs 3D inpainting of Neural Radiance Fields (NeRFs) by substituting a single reference view with a diffusion-generated target view, then optimizing the NeRF to match the new view while maintaining consistency with the remaining input views. It solves the problem of inserting or modifying objects in a 3D scene without requiring multi-view supervision for the target content. It differs from [Inpainting SDS Loss](11185-inpainting-sds-loss.md) by using a deterministic 2D inpainting prior (Stable Diffusion) followed by a bilateral solver for residual smoothing, rather than relying solely on gradient-based SDS guidance during optimization.

### [Masked Completion via GAN Inversion](20185-masked-completion-via-gan-inversion.md)
Masked Completion via GAN Inversion is a conditional sampling technique that completes a 3D radiance field by inverting a 2D discriminator-trained generator, using photometric error on non-masked regions to guide the optimization. It sets itself apart from sibling methods like [Nerfiller](14835-nerfiller.md) and [Reference-guided Controllable Inpainting of Neural Radiance Fields](15289-reference-guided-controllable-inpainting-of-neural-radiance-.md) by operating in the latent space of a pre-trained GAN generator (specifically EG3D) rather than optimizing a neural network's weights or using diffusion-based volumetric gradients directly.

### InFusion
InFusion is a 3D Gaussian inpainting method that integrates depth completion based on diffusion models [S1]. It selects a reference image for inpainting, employs its self-trained depth completion model to predict the depth map of the inpainting region, and projects it back into point clouds for accurate initial completion [S1]. It designs a progressive strategy to resolve complex cases that could not be completed from a single view [S1].

### 3D-Aware Inpainting
3D-Aware Inpainting is a technique that inpaints masked 2D regions of an image while simultaneously reconstructing the underlying 3D shape, using a 3D-aware diffusion model to find a latent 3D structure consistent with observed parts and plausible in masked regions [S2]. It conditions the denoising iterations on the known regions of the image, setting the previous timestep latent $x_{t-1}$ to the noised target pixels from the input image and mask in known regions, while sampling the unknown regions as usual based on the current timestep latent $x_t$ [S2]. It iterates the reverse diffusion process to converge on a latent 3D structure that satisfies both the observed image regions and the diffusion prior for the masked regions [S2]. Generated images may be blurrier than GAN-based methods, potentially reducing output fidelity [S2]. It requires training images to include camera extrinsics; triplanes are fixed in a global coordinate system, restricting generalization across object placements [S2]. It differs from RePaint [37] by replacing the 2D denoiser architecture with a 3D denoiser, enabling simultaneous 3D shape reconstruction alongside 2D inpainting [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization steps and the cost of rendering and diffusion sampling per step. Not given in the sources.
*   **Memory Complexity:** Depends on the resolution of the 3D representation and the diffusion model. Not given in the sources.
*   **Strengths:** Can handle arbitrary mask shapes and large missing regions [S1]. Leverages powerful 2D diffusion priors for high-quality texture generation [S1].
*   **Limitations:** May suffer from multi-view inconsistency if not explicitly constrained [S1]. Computationally expensive due to the need for multiple diffusion sampling steps per optimization iteration [S1].
*   **When to use:** When high-quality texture completion is required for large missing regions in a 3D scene, and multi-view consistency can be enforced through additional mechanisms (e.g., multi-view scoring) [S1].
*   **When not to use:** When real-time performance is critical, or when the missing regions are small and can be effectively handled by traditional interpolation or GAN-based methods [S1, S2].

## Background
Diffusion-Based 3D Inpainting emerged from the convergence of 3D scene representation techniques (like NeRF and 3DGS) and 2D image inpainting using diffusion models. Early methods like Inpaint3D [36] utilized self-trained 2D inpainting diffusion models with reconstruction and SDS losses [S1]. Subsequent works like Nerfiller [17] and MVIPNeRF [135] introduced multi-view consistency mechanisms [S1]. The approach has been extended to 3D Gaussian Splatting with methods like InFusion [16] and RefFusion [38], which integrate depth completion and discriminator losses [S1]. The concept of 3D-aware inpainting, as seen in RenderDiffusion [S2], involves using 3D denoisers to simultaneously reconstruct shape and texture, differing from 2D-only approaches like RePaint [37].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] RenderDiffusion: Image Diffusion for 3D Reconstruction, Inpainting and Generation, https://doi.org/10.1109/cvpr52729.2023.01213, fetched 2026-10-30
