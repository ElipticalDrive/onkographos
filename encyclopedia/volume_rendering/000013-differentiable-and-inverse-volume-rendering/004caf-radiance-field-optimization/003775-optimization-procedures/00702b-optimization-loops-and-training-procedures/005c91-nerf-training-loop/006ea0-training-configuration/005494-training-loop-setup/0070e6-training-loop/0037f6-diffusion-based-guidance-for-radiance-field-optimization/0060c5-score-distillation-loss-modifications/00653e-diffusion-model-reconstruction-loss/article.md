# Diffusion Model Reconstruction Loss

Also called: Modified LSDS with DM noise comparison, Unsuccessful Experiment: Diffusion Model Reconstruction Loss

## Overview
This method attempts to enforce fidelity to a specific input image during radiance field optimization by comparing the noise predictions of a diffusion model for a noisy rendered image against the noise predictions for a noisy version of the input image. It serves as an alternative to pixel-level reconstruction losses (like MSE) by operating in the latent noise space of a pre-trained diffusion model. It is distinct from standard Score Distillation Loss (SDS) [Diffusion-Based Guidance for Radiance Field Optimization](8117-diffusion-based-guidance-for-radiance-field-optimization.md) because it does not use a text prompt or unconditional prior to guide the generation, but rather uses the input image itself as the conditioning target for the noise comparison.

## How it works
**Inputs:**
*   Radiance field parameters $\Theta$.
*   Input reference image $I_{in}$ (or its latent representation).
*   Camera pose $p$ for rendering.
*   Frozen diffusion model $\epsilon_\theta$ (typically Stable Diffusion).
*   Noise scheduler parameters (timestep $t$).

**Outputs:**
*   Gradient $\nabla_\Theta \mathcal{L}_{DM-Rec}$ to update $\Theta$.

**Data Structures:**
*   **Rendered Image $I_{render}$:** Output of the differentiable renderer given $\Theta$ and $p$.
*   **Noisy Latent/Image $x_t$:** The rendered image (or its latent embedding) perturbed with Gaussian noise $\epsilon \sim \mathcal{N}(0,1)$ at timestep $t$.
*   **Noisy Input $y_t$:** The input image (or its latent embedding) perturbed with the same noise realization $\epsilon$ at timestep $t$.

**Steps:**
1.  **Render:** Compute the rendered image $I_{render}$ from the current radiance field $\Theta$ at camera pose $p$.
2.  **Encode (Optional):** If using a latent diffusion model (e.g., Stable Diffusion), encode $I_{render}$ and $I_{in}$ into latent space $z_{render}$ and $z_{in}$ using the VAE encoder $E$. [G: If the diffusion model operates directly on pixels, skip this step and use pixel values].
3.  **Perturb:** Sample noise $\epsilon$ and timestep $t$. Compute the noisy latent for the render:
    $$ z_t^* = \sqrt{\bar{\alpha}_t} z_{render} + \sqrt{1 - \bar{\alpha}_t} \epsilon $$
    Compute the noisy latent for the input:
    $$ y_t = \sqrt{\bar{\alpha}_t} z_{in} + \sqrt{1 - \bar{\alpha}_t} \epsilon $$
    Note: The same noise $\epsilon$ must be used for both to ensure a fair comparison of the *structure* of the noise prediction relative to the content.
4.  **Predict Noise:** Pass $z_t^*$ and $y_t$ through the frozen diffusion model $\epsilon_\theta$ (with empty text prompt or no conditioning if the model requires it, as per [S2] notation $\emptyset$).
    $$ \hat{\epsilon}_{render} = \epsilon_\theta(z_t^*, t, \emptyset) $$
    $$ \hat{\epsilon}_{input} = \epsilon_\theta(y_t, t, \emptyset) $$
5.  **Compute Loss:** Calculate the mean squared error between the two predicted noise vectors.
    $$ \mathcal{L}_{DM-Rec} = \| \hat{\epsilon}_{render} - \hat{\epsilon}_{input} \|_2^2 $$
6.  **Backpropagate:** Compute gradients $\nabla_\Theta \mathcal{L}_{DM-Rec}$ through the diffusion model (treating $\epsilon_\theta$ as differentiable for the purpose of gradient flow to the renderer, although $\epsilon_\theta$ weights are frozen) and the differentiable renderer.
7.  **Update:** Update $\Theta$ using an optimizer (e.g., Adam).

**Pitfalls:**
*   **Color/Texture Mismatch:** The method may reconstruct the general geometry and structure of the input image but fail to match exact colors or fine textures [S1].
*   **Noise Correlation:** Using different noise realizations for the render and input breaks the direct comparison of noise prediction errors. The same $\epsilon$ must be used.
*   **Conditioning:** If the diffusion model requires conditioning (e.g., text), using an empty prompt or a generic prompt is critical to avoid introducing external biases. [S2] uses $\emptyset$.

## Subtopics

### Reference View Score Distillation
A variant applied during NeuS mesh reconstruction to ensure consistency between the reconstructed geometry's rendering at the input viewpoint and the original input image using score distillation [S2]. It differs from the general DM Reconstruction Loss by being specifically integrated into a NeuS pipeline and often used alongside photometric losses. It emphasizes perceptual quality via patch-aware noise/latent-level comparison rather than pixel-level MSE [S2].
*   **Formula:**
    $$ \mathcal{L}_{RV} = \mathbb{E}_{\mathbf{t}, \epsilon} \left[ w(\mathbf{t}) \| \epsilon_{\psi}(\mathbf{z}_{\mathbf{t}}^*, \emptyset) - \epsilon_{\psi}(\mathbf{y}_{\mathbf{t}}, \emptyset) \|_2^2 \right] $$
    where $\mathbf{z}_t^* = R(\Theta, p^*)$ is the noisy latent of the NeuS rendering at input camera pose $p^*$, $\mathbf{y}_t$ is the noisy latent of the input view image, $\epsilon_\psi$ is the Stable Diffusion noise predictor, and $w(t)$ is a weighting function [S2].
*   **Procedure:** Add small noise (e.g., Stable Diffusion scheduler $t=200$) to NeRF render images, conduct denoising, and optimize NeuS parameters $\Theta$ by adding $\mathcal{L}_{RV}$ to the original photometric loss [S2].

## Operating characteristics
*   **Time Complexity:** Requires two forward passes through the diffusion model per optimization step (one for the render, one for the input), doubling the diffusion inference cost compared to standard SDS which uses one pass [S1, S2].
*   **Memory Complexity:** Requires storing the latent representations of both the rendered image and the input image, plus the noise tensors.
*   **Performance:** In experiments, this loss allowed reconstruction of the input image to a certain degree but failed to match exact input image colors or textures [S1]. It is considered an unsuccessful experiment for high-fidelity single-image reconstruction in the context of RealFusion [S1].
*   **Strengths:** Provides a perceptual prior that is more robust to pixel-level noise than MSE [S2].
*   **Limitations:** Does not guarantee photometric accuracy; may produce blurry or color-shifted results [S1].

## Background
This method was explored as part of the RealFusion pipeline for 360° reconstruction of objects from a single image [S1]. The authors investigated using the diffusion model within the reconstruction objective to guide the neural field towards the input image. They found that while it helped with structure, it was insufficient for color/texture fidelity. A similar concept, Reference View Score Distillation, was later applied in the context of NeuS mesh reconstruction from multi-view diffusers to leverage input view information for geometry and texture distillation [S2].

## Sources
[S1] RealFusion 360° Reconstruction of Any Object from a Single Image, https://doi.org/10.1109/cvpr52729.2023.00816, fetched 2026-10-30
[S2] Optimized View and Geometry Distillation from Multi-view Diffuser, https://arxiv.org/abs/2312.06198v4, fetched 2026-10-27
