# Rendering and Diffusion Prior Pipeline

Also called: LSDS Pipeline, Latent Space Diffusion Sampling

## Overview
The Rendering and Diffusion Prior Pipeline (LSDS) optimizes a neural radiance field from a single input image by simultaneously minimizing a reconstruction loss for the input view and a diffusion prior loss for randomly sampled novel viewpoints [S1]. It distinguishes itself from sibling methods by leveraging a frozen pre-trained diffusion model (Stable Diffusion) in the latent space to "dream up" missing views, enabling high-quality single-image 3D reconstruction without requiring multi-view inputs [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single input image $I_{in}$ and a text prompt $P$.
*   **Output:** Optimized neural radiance field parameters (density and color fields).

**Data Structures:**
*   **Radiance Field:** A 3D representation (e.g., Tri-Planes or MLP-based) parameterized by weights $\theta$. Features are stored in grids of dimensionality 2, stacked and fed to a 3-layer MLP with 64 hidden units [S1].
*   **Background Model:** A two-layer MLP taking viewing direction as input, purposefully weak to prevent trivial optimization via background [S1].
*   **VAE Encoder ($E_\phi$):** Maps 512px images to latent space [S1].
*   **Diffusion UNet:** Frozen pre-trained model for noise prediction [S1].

**Steps:**
1.  **Textual Inversion:** Generate a custom embedding token $\langle e \rangle$ from the input image using single-image textual inversion [S1].
2.  **Rendering:** Render a view from the current radiance field at 96px resolution [S1].
3.  **Upsampling:** Upsample the rendered image to 512px to match the Stable Diffusion input requirements [S1].
4.  **Latent Encoding:** Pass the upsampled image through the VAE encoder $E_\phi$ to obtain the latent representation $z$ [S1].
5.  **Noise Addition:** Sample a timestep $t \sim U(0.02, 0.98)$ and add noise to the latent $z$ [S1].
6.  **Diffusion Loss Calculation:** Compute the noise prediction error using the frozen UNet conditioned on the prompt $P$ with the custom token $\langle e \rangle$ and classifier-free guidance [S1].
7.  **Backpropagation:** Backpropagate the gradient through the VAE encoder $E_\phi$ and the radiance field parameters $\theta$. Do **not** backpropagate through the Stable Diffusion UNet [S1].
8.  **Optimization:** Update $\theta$ using the Adam optimizer [S1].

**Formulas:**
The total loss $L$ is a combination of the reconstruction objective $L_{rec}$ and the diffusion prior objective $L_{diff}$:
$$ L = L_{rec} + \lambda L_{diff} $$
Where $L_{diff}$ is derived from the noise prediction error of the diffusion model [S1].

**Parameters:**
*   **Render Resolution:** 96px [S1].
*   **Upsampled Resolution:** 512px [S1].
*   **Classifier-Free Guidance Strength:** 100 [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $1e-3$ [S1].
*   **Iterations:** 5000 [S1].

**Edge Cases and Pitfalls:**
*   **Guidance Strength Sensitivity:** Classifier-free guidance strength below 30 leads to geometric deformities; strengths above 30 produce good results [S1].
*   **Precision:** The VAE encoder gradients are used for optimization, not the UNet [S1].

## Subtopics

### Diffusion Prior Objective
An objective that uses a large pre-trained diffusion model to ensure the radiance field looks like the given object from randomly sampled novel viewpoints [S1]. It is chosen when single-image reconstruction requires strong geometric priors to fill in missing views [S1]. The process involves rendering novel views, passing them through the diffusion model to compute the prior objective, and optimizing the radiance field to minimize both the reconstruction objective (for the input view) and the prior objective (for novel views) simultaneously [S1].

## Operating characteristics
*   **Time Complexity:** Approximately 45 minutes on a single NVIDIA V100 GPU for 5000 iterations [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Enables high-quality 3D reconstruction from a single image by leveraging powerful 2D diffusion priors [S1].
*   **Limitations:** Requires careful tuning of classifier-free guidance strength to avoid geometric deformities [S1].
*   **When to use:** Single-image 3D reconstruction where multi-view data is unavailable [S1].
*   **When not to use:** When multi-view inputs are available, as other methods may be more efficient or accurate [S1].

## Background
Originated in the paper "RealFusion 360° Reconstruction of Any Object from a Single Image" [S1]. The method addresses the limitation of existing NeRF-like models that assume dozens or hundreds of views are available, extending them to single-image reconstruction by using a diffusion model to "dream up" missing views [S1]. It builds on techniques like textual inversion and latent space diffusion sampling [S1].

## Sources
[S1] RealFusion 360° Reconstruction of Any Object from a Single Image, https://doi.org/10.1109/cvpr52729.2023.00816, fetched 2026-10-30
