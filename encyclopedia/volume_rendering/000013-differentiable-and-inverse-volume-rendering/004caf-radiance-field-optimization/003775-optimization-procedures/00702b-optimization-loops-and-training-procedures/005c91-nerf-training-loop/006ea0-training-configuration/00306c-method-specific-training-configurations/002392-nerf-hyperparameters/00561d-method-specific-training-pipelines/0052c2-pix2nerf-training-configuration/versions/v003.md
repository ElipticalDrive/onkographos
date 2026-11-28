# Pix2NeRF Training Configuration

Also called: Pix2NeRF Training Pipeline, Unsupervised conditional $\pi$-GAN training

## Overview
Pix2NeRF is an unsupervised method that translates a single input image into a Neural Radiance Field (NeRF) by leveraging a pre-trained $\pi$-GAN generator [S1]. It solves the problem of single-image novel view synthesis without requiring multi-view supervision or test-time optimization [S1]. It is distinguished from its siblings, such as [BAD-NeRF](13523-bad-nerf.md), by its generative, unsupervised nature and its use of a discriminator-based encoder architecture rather than direct inverse optimization of scene parameters [S1].

## How it works
The method trains a conditional $\pi$-GAN where the generator produces NeRF parameters and the discriminator ensures photorealism. The training follows a progressive resolution strategy and uses the Adam optimizer for all components [S1].

**Inputs and Outputs**
*   **Input:** Single RGB image.
*   **Output:** NeRF parameters (density and color fields) and camera pose.

**Data Structures**
*   **Generator:** Produces NeRF representation.
*   **Discriminator:** Discriminates between real and generated views.
*   **Encoder:** Uses the discriminator architecture as a backbone with a tanh activation at the latent code head [S1].

**Numbered Steps**
1.  **Progressive Resolution Start:** Begin training at $32 \times 32$ resolution [S1].
2.  **Ray Sampling (32x32):** Sample 96 points per ray [S1].
3.  **Optimization (32x32):** Update parameters using Adam with specific learning rates (see Parameters) for 300k total iterations [S1].
4.  **Resolution Increase:** Increase training resolution to $64 \times 64$ [S1].
5.  **Ray Sampling (64x64):** Reduce samples to 24 points per ray [S1].
6.  **Optimization (64x64):** Continue optimization with updated learning rates for the remainder of the 300k iterations [S1].
7.  **Rendering:** Render final outputs at $128 \times 128$ resolution, regardless of the training resolution [S1].

**Parameters**
*   **Optimizer:** Adam [S1].
*   **Total Iterations:** 300,000 [S1].
*   **Batch Size:** 48 [S1].
*   **Learning Rates at 32x32:**
    *   Generator: $4 \times 10^{-4}$ [S1]
    *   Discriminator: $4 \times 10^{-5}$ [S1]
    *   Encoder: $4 \times 10^{-4}$ [S1]
*   **Learning Rates at 64x64:**
    *   Generator: $6 \times 10^{-5}$ [S1]
    *   Discriminator: $2 \times 10^{-4}$ [S1]
    *   Encoder: $2 \times 10^{-4}$ [S1]
*   **Note on LR Discrepancy:** Source [S1] lists generator LR as $4 \times 10^{-4}$ at 32x32, but later lists discriminator LR as $2 \times 10^{-4}$ and generator as $6 \times 10^{-5}$ at 64x64. The text explicitly states "learning rates 4e-5, 4e-4, and 4e-4 for generator, discriminator, and encoder" at 32x32, but then lists "2e-5, 2e-4, and 2e-4 for generator, discriminator, and encoder" at 64x64. We follow the explicit list order: Generator, Discriminator, Encoder.

**Edge Cases and Pitfalls**
*   **Super-Resolution Effect:** Training at $64 \times 64$ but rendering at $128 \times 128$ yields a super-resolution effect compared to models trained directly on $128 \times 128$ [S1].
*   **Computational Cost:** The training cost is approximately equivalent to obtaining a standard $\pi$-GAN model [S1].

## Subtopics

### Hybrid Inversion Optimization
A variant used for GAN inversion where pose parameters are optimized alongside latent codes.
*   **What it is:** An optimization step that inverts a GAN to fit a specific input image by optimizing latent code $w$, pose $q$, scale $s$, translation $t_2$, and noise $z_0$.
*   **What sets it apart:** It uses a dual discriminator (unconditional + conditional) instead of a single unconditional discriminator with L1/MSE loss, which avoids the need to balance multiple loss terms.
*   **When to choose:** When high-fidelity inversion of a pre-trained GAN is required, particularly for real-world datasets.
*   **Link:** [Hybrid Inversion Optimization](15628-hybrid-inversion-optimization.md) (Hypothetical link for the variant).

**Details for Hybrid Inversion Optimization:**
*   **Step:** After each iteration, reproject pose parameters onto valid constraints (ensure quaternion $q$ has unit length).
*   **Formula:** The objective function is:
    $$ \min_{w,q,s,t_2,z_0} \frac{1}{K} \sum_{k=1}^{K} \text{LPIPS}(c_{pred}^{[k]}, c_{gt}^{[k]}) $$
    Where $c_{pred}$ is the predicted image, $c_{gt}$ is the ground-truth image, $k$ is the augmentation index, $K=16$ augmentations are used, and LPIPS [70] measures the distance between VGG embeddings.
*   **Parameters:**
    *   Optimizer: Adam.
    *   Base Learning Rate: 0.02.
    *   Latent Code $w$ Learning Rate Gain: Optional, varies by experiment.
    *   Beta2: 0.95 (for faster reaction).
    *   Hyperparameters are not dynamically adjusted.

## Operating characteristics
*   **Time Complexity:** Training runs for 300k iterations [S1]. The computational cost is comparable to training a standard $\pi$-GAN [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Trained on CelebA [19] and CARLA [8] datasets [S1]. Produces decent 3D representations even under extreme poses and artifacts [S1].
*   **Scaling:** Uses progressive training from $32 \times 32$ to $64 \times 64$ [S1].
*   **Strengths:** Unsupervised, does not require test-time optimization, handles "in the wild" datasets lacking multi-view supervision [S1].
*   **Limitations:** Performance does not drop significantly below competition but is not explicitly stated to surpass it [S1].
*   **When to use:** Single-image novel view synthesis without multi-view data [S1].
*   **When not:** When multi-view supervision is available and test-time optimization is acceptable [S1].

## Background
Pix2NeRF was introduced in 2022 [S1]. It builds on the $\pi$-GAN framework [2] and addresses the limitations of existing few-shot NeRF methods that fail on "in the wild" datasets due to lack of multi-view supervision [S1]. It utilizes CelebA [19] and CARLA [8] for training and evaluation [S1].

## Sources
[S1] Pix2NeRF: Unsupervised Conditional $\pi$-GAN for Single Image to Neural Radiance Fields Translation, https://doi.org/10.1109/cvpr52688.2022.00395, fetched 2026-10-29
[S2] Shape, Pose, and Appearance from a Single Image via Bootstrapped Radiance Field Inversion, https://doi.org/10.1109/cvpr52729.2023.00427, fetched 2026-10-26
