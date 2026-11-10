# Inpainting SDS Loss
Also called: Inpainting-based Score Distillation Sampling, SDS loss with scene context

## Overview
Inpainting SDS Loss is a training objective for optimizing Neural Radiance Fields (NeRFs) that integrates a pre-trained 2D diffusion inpainting model to generate 3D objects that harmonize with an existing 3D scene [S1]. Unlike vanilla SDS loss, which often produces incomplete or poorly posed objects that fail to composite correctly, this method conditions the diffusion process on scene context (masked images and masks) to ensure complete geometry and proper alignment [S1]. It builds on [Diffusion-Based Guidance for Radiance Field Optimization](8117-diffusion-based-guidance-for-radiance-field-optimization.md) by replacing the standard noise prediction with an inpainting-aware prediction.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A NeRF parameterized by $\Theta$, a text prompt $y$, a current rendered image $I$, a mask $M$ defining the region to be inpainted, and the masked image $I_M$ (where the region $M$ is zeroed out or masked). The mask $M$ is derived by projecting a 3D bounding box of the target object into the camera viewpoint [S1].
*   **Outputs:** The gradient $\nabla_\Theta L_{SDS}$ used to update the NeRF parameters $\Theta$.

**Data Structures:**
*   **Mask $M$:** A binary mask of the same spatial dimensions as the rendered image, where pixels inside the projected 3D bounding box are 1 (or true) and others are 0.
*   **Masked Image $I_M$:** The rendered image $I$ with pixels in $M$ set to a neutral value (e.g., black or transparent) to provide context for the inpainting model.

**Algorithm:**
1.  **Sampling:** Randomly sample a diffusion time step $t$ and Gaussian noise $\epsilon \sim \mathcal{N}(0, I)$ [S1].
2.  **Noise Perturbation:** Create the noisy image $I_t$ by adding noise to the masked image $I_M$ according to the diffusion process at step $t$. Specifically, $I_t = \sqrt{\bar{\alpha}_t} I_M + \sqrt{1-\bar{\alpha}_t} \epsilon$ (standard DDPM forward process notation, implied by the use of $I_t$ in the loss) [S1].
3.  **Inpainting Prediction:** Pass $I_t$, the text prompt $y$, the masked image $I_M$, and the mask $M$ into the frozen inpainting diffusion model $\epsilon_\phi$. The model predicts the noise $\epsilon_\phi(I_t; y, I_M, M, t)$ [S1].
4.  **Gradient Computation:** Compute the loss gradient with respect to the NeRF parameters $\Theta$ using the following formula [S1]:

    $$
    \nabla_\Theta L_{SDS} = \mathbb{E}_{t,\epsilon,C} \left[ \omega(t) \cdot \left( \epsilon_\phi(I_t; y, I_M, M, t) - \epsilon \right) \frac{\partial I_t}{\partial \Theta} \right]
    $$

    Where:
    *   $\omega(t)$ is a weighting function for the time step $t$ [S1].
    *   $\frac{\partial I_t}{\partial \Theta}$ is the gradient of the noisy rendered image with respect to the NeRF parameters, computed via backpropagation through the differentiable renderer [S1].
    *   $C$ represents the camera parameters [S1].

5.  **Update:** Update $\Theta$ using the computed gradient (e.g., via Adam optimizer) [S1].

**Parameters:**
*   **Mask $M$:** Derived by projecting the 3D bounding box of the target object into the current camera view [S1].
*   **Text Prompt $y$:** Describes the object to be generated [S1].

**Edge Cases and Pitfalls:**
*   **Mask Accuracy:** The quality of the inpainting depends heavily on the accuracy of the projected 3D bounding box. Incorrect projection leads to misaligned generation [S1].
*   **Saturation:** Without additional constraints, generated objects may exhibit over-saturation. A saturation loss is often applied alongside SDS to regulate color intensity [S1].
*   **Geometry:** To ensure coherent 3D geometry, sparsity loss and opacity loss are typically employed in conjunction with the SDS loss to encourage sparse, well-defined object boundaries [S1].

## Subtopics

### MVIP-NeRF
MVIP-NeRF is a variant that extends inpainting SDS to multiview-consistent NeRF inpainting [S2]. Unlike standard inpainting SDS which may operate per-view, MVIP-NeRF jointly optimizes across multiple views to ensure consistency [S2]. It employs a multi-view SDS formulation to enhance consistency under large view changes, distills normal maps to enforce geometric consistency, and performs RGB reconstruction in visible areas alongside inpainting in masked regions [S2].

## Operating characteristics
*   **Time Complexity:** Comparable to vanilla SDS, as it involves one forward pass through the diffusion model and one backpropagation through the NeRF per iteration [S1].
*   **Memory Complexity:** Requires storing the NeRF parameters, the current rendered images, and the diffusion model weights (frozen) [S1].
*   **Strengths:** Generates complete objects with proper poses that seamlessly integrate with the scene, significantly outperforming vanilla SDS in compositional quality (e.g., CLIP scores >74 vs <63 for baselines) [S1].
*   **Limitations:** Relies on the accuracy of the 3D bounding box projection for mask generation [S1]. May require additional losses (saturation, sparsity) to prevent visual artifacts [S1].
*   **When to use:** When generating new 3D objects into an existing 3D scene where alignment and completeness are critical [S1].
*   **When not to use:** When the scene context is not available or the bounding box cannot be reliably determined [S1].

## Background
Inpainting SDS Loss was introduced in GO-NeRF (2025) to address the limitation of vanilla SDS (e.g., DreamFusion) in generating objects that align with existing 3D scenes [S1]. Vanilla SDS often produces incomplete parts (e.g., only a cat's head) that fail to composite [S1]. The method draws inspiration from 2D image inpainting, using a pre-trained diffusion model conditioned on text and scene context (mask and masked image) to guide 3D generation [S1]. MVIP-NeRF (2024) later adapted this concept for multiview consistency [S2].

## Sources
[S1] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
