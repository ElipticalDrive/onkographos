# Conditional Adversarial Autoencoder

Also called: Auto-encoder Variant, CAE, Conditional Adversarial Autoencoder for Volume Compression

## Overview
The Conditional Adversarial Autoencoder (CAE) is a reconstruction pipeline variant that adapts the $\pi$-GAN architecture by dropping the explicit latent code injection step [S1]. Instead of sampling or optimizing a latent vector, it trains only the reconstruction and conditional adversarial objectives directly [S1]. Unlike full GAN inversion pipelines that rely on a latent connection step, this approach attempts direct conditional generation but is noted to suffer from optimization difficulties that cause visible 3D inconsistencies despite decent reconstruction quality [S1].

## How it works
**Inputs and Outputs:**
- Input: A single source image serving as the condition $c$.
- Output: A reconstructed 3D volume or NeRF representation.

**Data Structures:**
- The pipeline utilizes a generator network $G$ and a discriminator $D$ derived from the $\pi$-GAN architecture [S1].
- The latent space is explicitly removed from the pipeline; no latent code $z$ is stored or passed through during training or inference [S1].

**Training Objectives:**
The model is trained using two primary objectives [S1]:
1. Reconstruction objective: Measures the difference between the generated volume and the ground truth volume.
2. Conditional adversarial objective: Encourages the generated volume to be indistinguishable from real volumes given the condition.

**Steps:**
1. Utilize the $\pi$-GAN architecture for the generator and discriminator [S1].
2. Drop the latent space from the pipeline [S1].
3. Train the network using only the reconstruction and conditional adversarial objectives [S1].

**Parameters:**
- Loss weighting coefficients: Not given in the sources.

**Edge Cases and Pitfalls:**
- Optimization difficulty: Removing the latent space leads to visible 3D inconsistencies in the output, even when reconstruction metrics appear decent, indicating that the remaining objectives are difficult to optimize jointly [S1].
- Blurriness: Removing the GAN inversion step entirely without proper adaptation results in blurry and uncanny visual outputs, as this step normally connects $\pi$-GAN training and reconstruction [S1].

## Subtopics

### CCA Bitrate Profiles
Configurable compression settings for canonical geometry and appearance attributes in the CCA framework, acting as a decoupled bitrate knob that trades storage for PSNR/SSIM [S2]. It allows selection between Low, Mid, and Default profiles by adjusting specific bit budgets [S2].
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
- **Measured Performance:** On the N3DV dataset (6-scene average), the CCA variant achieves 32.06 dB PSNR and 0.943 SSIM at a default size of 29.67 MB [S2]. A Low profile reduces this to 23.46 MB with 31.92 dB PSNR [S2].
- **Scaling:** Long-sequence stress tests on the flame_salmon scene show gradual degradation as sequence length increases from 300 to 1200 frames, with comparable methods dropping ~0.50 dB PSNR [S2].
- **Strengths:** Decoupled bitrate control allows storage optimization without retraining the deformation pathway [S2].
- **Limitations:** Visible 3D inconsistencies may occur due to optimization difficulties when the latent space is removed [S1].

## Background
The Conditional Adversarial Autoencoder concept originates from the Pix2NeRF paper, which explores unsupervised conditional $\pi$-GANs for single-image to NeRF translation [S1]. The specific "Auto-encoder" variant was proposed as an ablation study to investigate the role of the latent space in GAN-based volume generation [S1]. The CCA Bitrate Profiles are detailed in the CC-4DGS paper, which focuses on computational deformation and point-cloud compression for dynamic Gaussian Splatting [S2].

## Sources
[S1] Pix2NeRF: Unsupervised Conditional $\pi$-GAN for Single Image to Neural Radiance Fields Translation, https://doi.org/10.1109/cvpr52688.2022.00395, fetched 2026-10-29
[S2] CC-4DGS: Computational Deformation and Point-Cloud Compression for Storage-Efficient Dynamic Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3721136, fetched 2026-10-27
