# Conditional Adversarial Autoencoder

Also called: Auto-encoder Variant, CAE, Conditional Adversarial Autoencoder for Volume Compression

## Overview
The Conditional Adversarial Autoencoder (CAE) is a neural network compression method that adapts a pre-trained conditional Generative Adversarial Network (GAN) into a reconstruction pipeline by removing the explicit latent code injection step [S1]. It solves the problem of generating high-fidelity 3D volumes from single images by leveraging the GAN's learned prior for adversarial regularization while relying on an autoencoder structure for direct reconstruction [S1]. Unlike standard autoencoders that optimize only reconstruction loss, or full GANs that require latent sampling, the CAE trades off the flexibility of latent space exploration for a more direct, albeit potentially inconsistent, optimization path [S1]. It is distinct from the Latent Voxel Auto-Encoder, which explicitly maintains and optimizes a discrete latent voxel grid for storage efficiency [17589-latent-voxel-auto-encoder.md].

## How it works
The CAE variant is derived from the $\pi$-GAN architecture used in Pix2NeRF [S1]. It operates by training the generator network to map input conditions directly to output volumes without an intermediate latent vector $z$ being fed into the generator during inference or training [S1].

**Inputs and Outputs:**
- Input: A single source image (condition).
- Output: A reconstructed 3D volume (e.g., NeRF representation or voxel grid).

**Data Structures:**
- The generator network $G$ takes the condition $c$ (image) as input.
- The discriminator $D$ evaluates the realism of the generated volume $V_{gen}$ against real volumes $V_{real}$.
- No latent code $z$ is stored or passed through the pipeline [S1].

**Training Objectives:**
The model is trained using two primary objectives [S1]:
1.  **Reconstruction Loss ($L_{rec}$):** Measures the difference between the generated volume and the ground truth volume.
2.  **Conditional Adversarial Loss ($L_{adv}$):** Encourages the generated volume to be indistinguishable from real volumes given the condition.

The total loss is a weighted sum:
$$ L_{total} = L_{rec} + \lambda_{adv} L_{adv} $$

**Steps:**
1.  Initialize the generator $G$ and discriminator $D$ from the pre-trained $\pi$-GAN architecture [S1].
2.  Remove the latent space injection mechanism from the generator's forward pass [S1].
3.  For each training batch:
    a. Generate volume $V_{gen} = G(c)$ from the input condition $c$ [S1].
    b. Compute reconstruction loss $L_{rec}$ between $V_{gen}$ and ground truth $V_{real}$ [S1].
    c. Compute adversarial loss $L_{adv}$ by distinguishing $V_{gen}$ from $V_{real}$ conditioned on $c$ [S1].
    d. Update $G$ and $D$ using the combined loss [S1].

**Parameters:**
- $\lambda_{adv}$: Weight for the adversarial loss. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Optimization Difficulty:** Removing the latent space can lead to visible 3D inconsistencies in the output, even if the reconstruction quality (e.g., PSNR) appears decent [S1]. This suggests that the remaining objectives are difficult to optimize jointly without the stabilizing effect of the latent prior [S1].
- **Blurriness:** If the GAN inversion step (used in the full Pix2NeRF pipeline) is removed entirely without proper adaptation, results may become blurry and uncanny [S1]. The CAE approach specifically drops the latent space but retains the adversarial framework, which may still suffer from these issues if not carefully tuned [S1].

## Subtopics

### CCA Bitrate Profiles
Configurable compression settings for canonical geometry and appearance attributes in the CCA (Canonical Compression Autoencoder) framework, acting as a decoupled bitrate knob that trades storage for PSNR/SSIM [S2]. It allows users to select between Low, Mid, and Default profiles by adjusting specific bit budgets [S2].
- **Parameters:**
    - AE latent bits: 10 (Low), 12 (Mid), 14 (Default) [S2].
    - Codebook entries: 64 (Low), 128 (Mid), 256 (Default) [S2].
    - Scalar attr bits: 8 (Low), 10 (Mid), 12 (Default) [S2].
    - Dynamic bits: 10 (Low), 12 (Mid), 16 (Default) [S2].
- **Performance (N3DV 6-scene average):**
    - Low: 23.46 MB, 31.92 dB PSNR, 0.942 SSIM [S2].
    - Mid: 26.71 MB, 32.04 dB PSNR, 0.943 SSIM [S2].
    - Default: 29.67 MB, 32.06 dB PSNR, 0.943 SSIM [S2].
- **Trade-off:** Lowering the profile from Default to Low reduces total model size by ~21% while losing only ~0.14 dB PSNR [S2].

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources.
- **Measured Performance:** On the N3DV dataset (6-scene average), the CCA variant (associated with CAE configurations) achieves 32.06 dB PSNR and 0.943 SSIM at a default size of 29.67 MB [S2]. A Low profile reduces this to 23.46 MB with 31.92 dB PSNR [S2].
- **Scaling:** Long-sequence stress tests on the flame_salmon scene show gradual degradation as sequence length increases from 300 to 1200 frames (e.g., -0.50 dB PSNR drop for comparable methods) [S2].
- **Strengths:** Decoupled bitrate control allows storage optimization without retraining the deformation pathway [S2].
- **Limitations:** Visible 3D inconsistencies may occur due to optimization difficulties when the latent space is removed [S1].

## Background
The Conditional Adversarial Autoencoder concept originates from the Pix2NeRF paper, which explores unsupervised conditional $\pi$-GANs for single-image to NeRF translation [S1]. The specific "Auto-encoder" variant was proposed as an ablation study to investigate the role of the latent space in GAN-based volume generation [S1]. The CCA Bitrate Profiles are detailed in the CC-4DGS paper, which focuses on computational deformation and point-cloud compression for dynamic Gaussian Splatting [S2].

## Sources
[S1] Pix2NeRF: Unsupervised Conditional $\pi$-GAN for Single Image to Neural Radiance Fields Translation, https://doi.org/10.1109/cvpr52688.2022.00395, fetched 2026-10-29
[S2] CC-4DGS: Computational Deformation and Point-Cloud Compression for Storage-Efficient Dynamic Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3721136, fetched 2026-10-27
