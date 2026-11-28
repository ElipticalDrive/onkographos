# Method-Specific Training Pipelines
## Overview
This family covers specialized training configurations and optimization loops for Neural Radiance Fields (NeRF) that deviate from standard baselines to handle specific input conditions or scene structures [G]. Unlike generic NeRF training, these pipelines integrate domain-specific constraints—such as physical exposure models for motion blur, generative adversarial frameworks for single-image synthesis, or coordinate contraction for large-scale stratified environments—directly into the optimization loop [BAD-NeRF](13523-bad-nerf.md), [Pix2NeRF Training Configuration](14803-pix2nerf-training-configuration.md), and [Strata-NeRF Training Configuration](#strata-nerf-training-configuration) [G].

## Shared principles
All methods in this family optimize NeRF parameters via gradient descent using the Adam optimizer [G]. They rely on differentiable volume rendering to backpropagate gradients from image space to the scene representation [G], and sample points along camera rays to query density and color fields [G]. The number of samples and sampling strategy are hyperparameters tuned per method [G]. Each pipeline defines a specific loss function and optimizes auxiliary variables alongside the NeRF [G].

## Subtopics

### [BAD-NeRF](13523-bad-nerf.md)
BAD-NeRF reconstructs 3D scenes and recovers camera poses from motion-blurred images by explicitly modeling the physical image formation process during the camera's exposure time [BAD-NeRF](13523-bad-nerf.md). It differs from its siblings by jointly optimizing the NeRF parameters and the continuous camera motion trajectory (start and end poses) for each image, synthesizing blurred images by averaging interpolated sharp intermediate views [BAD-NeRF](13523-bad-nerf.md). Choose this method when inputs are motion-blurred and accurate pose estimation is required [G].

### [Pix2NeRF Training Configuration](14803-pix2nerf-training-configuration.md)
Pix2NeRF is an unsupervised method for translating a single image into a NeRF by leveraging a pre-trained $\pi$-GAN [Pix2NeRF Training Configuration](14803-pix2nerf-training-configuration.md). It differs from its siblings by relying on a generative adversarial framework with a discriminator and encoder, rather than explicit geometric constraints or multi-view consistency losses [Pix2NeRF Training Configuration](14803-pix2nerf-training-configuration.md). It achieves a super-resolution effect by training at 64×64 resolution while rendering at 128×128 [Pix2NeRF Training Configuration](14803-pix2nerf-training-configuration.md). Choose this method for single-image 3D reconstruction where multi-view data is unavailable [G].

### Strata-NeRF Training Configuration
Strata-NeRF optimizes NeRFs for stratified scenes (e.g., multi-level buildings) using a specific configuration derived from Mip-NeRF 360 [S1]. It uses the "360.gin" configuration file, which employs a contract function to map input coordinates onto a ball of radius 2, handling large scenes better than the "blender 256.gin" configuration [S1]. It uses a vector codebook in a "Latent Generator" network [S1]. Choose this method for large-scale, multi-level synthetic or real-world scenes where standard NeRF configurations fail to capture global structure [G].
- step: Learning rate schedule uses log-linear interpolation from 0.002 down to 0.00002 at maximum steps, with 512 warmup steps [S1].
- parameter: Adam optimizer hyperparameters are $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-6}$, and initial learning rate $= 0.002$ [S1]. Loss weighting for the Latent Generator is $\lambda_2 = 0.1$ [S1]. Codebook sizes tested are 512, 1024, and 4096, with size 1024 yielding the best overall PSNR/SSIM/LPIPS across Coffee Shop, Bhutanese House, and Buddhist Temple datasets [S1].
- pitfall: Distortion loss from Mip-NeRF 360 is disabled for Blender datasets to prevent artifacts, following the original authors' recommendation [S1].
- comparison: The method uses the “360.gin” configuration instead of “blender 256.gin” because the contract function handles large scenes better, yielding higher PSNR/SSIM and lower LPIPS [S1]. Increasing codebook size to 4096 does not improve metrics and may slightly degrade performance, while 512 underfits, making 1024 the optimal balance between capacity and generalization [S1].

## Background
Method-specific training pipelines were developed to address limitations of standard NeRF training when applied to non-ideal inputs or complex scene geometries [G]. BAD-NeRF was introduced to handle the physical reality of camera exposure times, which standard NeRF ignores [BAD-NeRF](13523-bad-nerf.md). Pix2NeRF was developed to enable single-image 3D reconstruction without multi-view supervision [Pix2NeRF Training Configuration](14803-pix2nerf-training-configuration.md). Strata-NeRF adapted Mip-NeRF 360's architecture to handle large-scale stratified environments using a coordinate contraction function [S1].

## Sources
[S1] Strata-NeRF : Neural Radiance Fields for Stratified Scenes, https://arxiv.org/abs/2308.10337v1, fetched 2026-10-26
