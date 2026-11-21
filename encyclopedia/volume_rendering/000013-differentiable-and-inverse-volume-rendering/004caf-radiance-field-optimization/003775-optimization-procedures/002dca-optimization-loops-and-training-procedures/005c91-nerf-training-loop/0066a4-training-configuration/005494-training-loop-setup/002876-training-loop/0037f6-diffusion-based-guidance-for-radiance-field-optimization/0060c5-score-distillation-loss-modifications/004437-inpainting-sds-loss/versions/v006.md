# Inpainting SDS Loss

Also called: Inpainting-based Score Distillation Sampling, SDS loss with scene context

## Overview
Inpainting SDS Loss (LiSDS) is a modification of the Score Distillation Sampling (SDS) loss that uses a pre-trained or fine-tuned 2D inpainting diffusion model to generate 3D content conditioned on existing scene context [S1, unofficial][S2]. It solves the problem of incomplete geometry and poor compositional harmony when inserting objects into existing scenes, which vanilla SDS often fails to achieve by generating isolated parts (e.g., only a head) that do not align with the background [S2]. Unlike vanilla SDS, which conditions only on a text prompt, LiSDS conditions the diffusion model on a masked version of the rendered image and a binary mask, ensuring the generated object seamlessly integrates with the established scene [S2].

## How it works
The method operates within the standard differentiable rendering loop [Parent](18224-score-distillation-loss-modifications.md). It replaces the standard noise prediction $\epsilon_\theta(x_t; y, t)$ with an inpainting-aware noise prediction $\epsilon_\phi(x_t; y, I_M, M, t)$, where the model predicts noise for the masked region while preserving the unmasked region $I_M$ [S2].

**Inputs and Outputs:**
*   **Input:** Current radiance field parameters $\Theta$, camera pose, text prompt $y$, binary mask $M$, and the rendered image $I$ (or latent $z_0$).
*   **Output:** Gradient $\nabla_\Theta L$ to update $\Theta$.

**Data Structures:**
*   **Mask $M$:** A binary mask of the same spatial dimensions as the rendered image. It is derived by projecting the 3D bounding box of the target object into the current camera viewpoint [S2]. Pixels inside the box are 1 (masked/inpainting region), and pixels outside are 0 (context region) [S2].
*   **Masked Image $I_M$:** The rendered image $I$ where pixels outside the mask $M$ are preserved, and pixels inside $M$ are typically zeroed out or replaced with a neutral value before adding noise, depending on the specific inpainting model implementation [S2].

**Steps:**
1.  **Render:** Render the current image $I$ from $\Theta$ using the current camera pose.
2.  **Mask Generation:** Compute the binary mask $M$ by projecting the 3D bounding box of the object being optimized onto the image plane [S2].
3.  **Masked Input Preparation:** Create the masked image $I_M$ by applying $M$ to $I$. The specific handling of the masked region (e.g., zeroing) depends on the diffusion model's requirements [S2].
4.  **Noise Addition:** Sample a timestep $t$ and Gaussian noise $\epsilon$. Add noise to $I_M$ to obtain the noisy sample $x_t$ [S2].
5.  **Inpainting Noise Prediction:** Query the frozen inpainting diffusion model $\epsilon_\phi$ with inputs: noisy image $x_t$, text embedding $y$, masked image $I_M$, mask $M$, and timestep $t$. The model outputs the predicted noise $\hat{\epsilon} = \epsilon_\phi(x_t; y, I_M, M, t)$ [S2].
6.  **Gradient Computation:** Compute the SDS gradient using the residual between the predicted noise and the actual noise, weighted by $\omega(t)$, and backpropagate through the renderer [S2].

**Formula:**
The gradient for the radiance field parameters $\Theta$ is:
$$ \nabla_\Theta L_{SDS} = \mathbb{E}_{t,\epsilon,C} \left[ \omega(t) \cdot (\epsilon_\phi(I_t; y, I_M, M, t) - \epsilon) \frac{\partial I_t}{\partial \Theta} \right] $$
where:
*   $\epsilon_\phi$ is the inpainting diffusion model [S2].
*   $I_t$ is the noise-perturbed masked image at timestep $t$ [S2].
*   $y$ is the text prompt embedding [S2].
*   $I_M$ is the masked image (context) [S2].
*   $M$ is the binary mask derived from the projected 3D bounding box [S2].
*   $\epsilon$ is the sampled Gaussian noise [S2].
*   $\omega(t)$ is the weighting function [Parent](18224-score-distillation-loss-modifications.md).

**Alternative Formulation (LiSDS):**
In some implementations (e.g., DITTO-NeRF), the loss is formulated as a mean squared error between the predicted noise and the actual noise, used to optimize latent vectors or rendered latents [S1, unofficial]:
$$ L_{iSDS}(\phi, g(\theta)) = \mathbb{E}_{t,\epsilon} \left[ ||\epsilon_\phi(x_t; y, t, M) - \epsilon_t||^2_2 \right] $$
where $x_t$ is the noise image, $\epsilon_t$ is the actual noise, and $M$ is the binary mask [S1, unofficial]. This is often combined with a Reliability-guided loss $L_R$ for in-boundary regions and sparsity loss $L_{sp}$ [S1, unofficial].

**Parameters:**
*   **Mask $M$:** Derived from projecting the 3D bounding box [S2].
*   **Training Schedule:** In some variants, a refinement step is applied during the last 10% of training, where random foreground patches are masked and optimized with LiSDS to improve texture continuity [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Mask Precision:** The accuracy of the projected 3D bounding box directly affects the quality of the inpainting. Incorrect projection leads to misaligned generation [S2].
*   **Context Leakage:** If the mask $M$ is too small or the inpainting model does not properly condition on $I_M$, the generated object may not harmonize with the scene [S2].
*   **Incomplete Generation:** Vanilla SDS may generate incomplete objects; LiSDS mitigates this by conditioning on the scene context [S2].

## Subtopics

### [MVIP-NeRF](11931-mvip-nerf.md)
MVIP-NeRF is a diffusion-based method for multiview-consistent inpainting of Neural Radiance Fields (NeRFs). It solves the problem of reconstructing missing geometry and texture in masked regions of a scene while maintaining consistency across multiple camera views. Unlike prior approaches that rely on independent 2D inpainting per view—often resulting in inconsistencies and poor geometry—MVIP-NeRF jointly optimizes across views. It employs Score Distillation Sampling (SDS) with a text-conditioned diffusion model to guide inpainting in masked regions, alongside RGB reconstruction in visible areas.

### RefFusion
RefFusion inpaints one reference image using an inpainting Latent Diffusion Model (LDM) on both global and locally cropped versions of the reference view [S3]. It utilizes SDS loss to distill learned priors from the adapted LDM into the scene, incorporating a discriminator loss and depth loss to improve quality [S3]. Quantitatively, RefFusion surpassed all baselines in LPIPSdir (0.4283) compared to NeRF-In (0.4884), SPIn-NeRF (SD) (0.5701), SPIn-NeRF (LaMa) (0.4654), Inpaint3D (0.5150), Reference-guided inpainting (SDV2) (0.4532), and Reference-guided inpainting (SDXL) (0.4453) [S3].

### Random Patch Refinement
A refinement step applied during the last 10% of training to improve texture and color continuity by masking foreground patches and optimizing them with LiSDS, excluding the Reliability-guided loss ($L_R$) [S1, unofficial]. During this phase, if an in-boundary view is sampled, $L_R$ is excluded, and a random patch mask is applied to the foreground to refine the corresponding part using LiSDS [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Similar to vanilla SDS, as it involves one forward pass through the diffusion model per gradient step. However, the need to project 3D bounding boxes to generate masks adds a small overhead [S2].
*   **Memory Complexity:** Requires storing the mask $M$ and potentially the masked image $I_M$ [S2].
*   **Performance:** Generates complete objects with proper poses that seamlessly integrate with the scene, outperforming vanilla SDS which often generates incomplete parts [S2].
*   **Strengths:** Ensures harmonized composition and complete geometry by leveraging scene context [S2].
*   **Limitations:** Requires accurate 3D bounding box estimation for mask generation [S2]. May require fine-tuning of the diffusion model for specific inpainting tasks [S1, unofficial].
*   **When to use:** When inserting objects into existing scenes or inpainting missing regions in NeRFs [S1, unofficial][S2].
*   **When not to use:** When generating objects from scratch without scene context, vanilla SDS may be sufficient [S2].

## Background
Inpainting SDS Loss was introduced to address the limitations of vanilla SDS in scene composition tasks. Vanilla SDS often generates incomplete objects that fail to composite well with the background [S2]. By conditioning the diffusion model on masked images and masks, LiSDS leverages the generative priors of 2D inpainting models to guide 3D optimization [S2]. The concept has been explored in works such as DITTO-NeRF [S1, unofficial] and GO-NeRF [S2], and is related to methods like RefFusion [S3] and MVIP-NeRF [S1, unofficial][S2].

## Sources
[S1] DITTO-NeRF: Diffusion-based Iterative Text To Omni-directional 3D Model, https://arxiv.org/abs/2304.02827v1, fetched 2026-10-26 (unofficial)
[S2] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
