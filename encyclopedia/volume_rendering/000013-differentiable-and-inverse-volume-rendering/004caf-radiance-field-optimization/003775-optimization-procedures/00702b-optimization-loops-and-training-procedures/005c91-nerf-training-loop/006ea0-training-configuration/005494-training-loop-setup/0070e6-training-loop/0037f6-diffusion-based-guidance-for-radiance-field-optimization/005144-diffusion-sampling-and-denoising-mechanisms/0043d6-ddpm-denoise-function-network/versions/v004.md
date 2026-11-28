# DDPM Denoise Function Network

Also called: 3D Convolutional Denoiser, DDPM Denoiser

## Overview
The DDPM Denoise Function Network is a neural network architecture used within diffusion-based radiance field optimization to predict the noise component $\epsilon$ or the clean latent field $x_0$ at a given timestep $t$. It serves as the core generative prior for synthesizing 3D volumetric representations (such as voxel grids) from 2D image conditions. It differs from sibling methods like [DDIM Sampling with Classifier-Free Guidance](8118-ddim-sampling-with-classifier-free-guidance.md) by its specific 3D convolutional topology designed for volumetric data, rather than standard 2D image architectures or implicit coordinate-based networks.

## How it works
**Inputs and Outputs:**
*   Input: A noisy latent volume $x_t$ (voxel grid) and a timestep embedding $t$.
*   Output: Predicted noise $\epsilon_\theta(x_t, t)$ or denoised estimate $\hat{x}_0$.

**Data Structures:**
The network is an 8-layer 3D convolutional encoder-decoder [S1, unofficial].
*   **Layer Widths:** $\{32, 64, 128, 256, 256, 128, 64, 32\}$ [S1, unofficial].
*   **Resolution Mapping:**
    *   Input resolution: $32 \times 32 \times 32$ [S1, unofficial].
    *   Encoder (Layers 1-4): Reduces resolution from 32 to 4 using stride 2 convolutions [S1, unofficial].
    *   Bottleneck: Layer 4 output has width 256 and resolution $4 \times 4 \times 4$ [S1, unofficial].
    *   Decoder (Layers 5-8): Recovers resolution from 4 to 32 using 3D transposed convolutions with stride 2 [S1, unofficial].
    *   Output resolution: $32 \times 32 \times 32$ [S1, unofficial].

**Algorithm:**
1.  **Forward Pass:** Pass $x_t$ and $t$ through the 8-layer 3D CNN. The first four layers downsample spatially; the last four upsample spatially.
2.  **Noise Prediction:** The network outputs $\epsilon_\theta(x_t, t)$.
3.  **Denoised Estimate:** Compute $\hat{x}_0$ using the parent's formula:
    $$ \hat{x}_0 = \frac{x_t - \sqrt{1 - \bar{\alpha}_t} \epsilon_\theta(x_t, t)}{\sqrt{\bar{\alpha}_t}} $$
4.  **Rendering-Guided Update (Image-to-Volume):** For conditional synthesis, the predicted denoised field $\tilde{f}_0^t$ is updated to minimize photometric error against a target image $I$ [S2].
    *   Render the current estimate $\tilde{f}_0^t$ from a given pose to get $\tilde{I}_t$ [S2].
    *   Compute gradient $\nabla_{\tilde{f}_0^t} \mathcal{L}(\tilde{I}_t, I)$ via differentiable volumetric rendering [S2].
    *   Update the latent code: $\tilde{f}_0^t \leftarrow \tilde{f}_0^t - \lambda \nabla_{\tilde{f}_0^t} \mathcal{L}(\tilde{I}_t, I)$ [S2].
    *   $\lambda$ is a small guidance weight [S2].

**Parameters:**
*   Learning rate for latent code update: $10^{-2}$ [S2].
*   L2 regularization weight on code: $5 \times 10^{-2}$ [S2].
*   Guidance weight $\lambda$: Small value (exact value not specified, described as "small guidance weight") [S2].

**Edge Cases:**
*   Precision: Volumetric rendering gradients can be noisy; small $\lambda$ is required to prevent divergence [S2].
*   Boundaries: The network assumes a fixed $32^3$ voxel grid [S1, unofficial].

## Subtopics

### Classifier Guidance for Image-to-Volume Synthesis
A variant where the diffusion process is steered by gradients from a classifier or rendering loss rather than just text/image embeddings. It conditions the trained radiance field diffusion model by steering the sampling process using the gradient of the photometric error between the rendered view and the target image [S2]. Choose this when synthesizing 3D volumes from specific posed and segmented 2D images rather than text prompts [S2].

## Operating characteristics
*   **Memory:** High memory usage due to $32^3$ voxel grid storage and 3D convolutions. The bottleneck features are $256 \times 4^3$ [S1, unofficial].
*   **Time:** Slower than 2D diffusion due to 3D convolution operations.
*   **Strengths:** Explicit volumetric representation allows direct differentiable rendering for guidance [S2].
*   **Limitations:** Fixed resolution ($32^3$) limits detail; memory intensive for high-res scenes [S1, unofficial].

## Background
Originated in "DiffRF: Rendering-Guided 3D Radiance Field Diffusion" [S2] and "Neural Volumetric Mesh Generator" [S1, unofficial]. It applies DDPM principles to 3D radiance fields, using a 3D CNN denoiser to generate voxel-based representations [S1, unofficial][S2].

## Sources
[S1] Neural Volumetric Mesh Generator, https://arxiv.org/abs/2210.03158v1, fetched 2026-10-27 (unofficial)
[S2] DiffRF: Rendering-Guided 3D Radiance Field Diffusion, https://doi.org/10.1109/cvpr52729.2023.00421, fetched 2026-10-26
