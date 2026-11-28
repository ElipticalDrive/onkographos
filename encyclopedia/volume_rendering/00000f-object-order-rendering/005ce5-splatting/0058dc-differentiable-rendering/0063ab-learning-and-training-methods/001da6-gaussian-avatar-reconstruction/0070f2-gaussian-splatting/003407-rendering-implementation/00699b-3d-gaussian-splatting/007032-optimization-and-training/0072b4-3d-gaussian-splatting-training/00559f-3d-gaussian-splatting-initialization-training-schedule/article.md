# 3D Gaussian Splatting Initialization & Training Schedule

Also called: LiDAR-guided 3DGS Training, Pseudo-view Guided 3DGS Training

## Overview
This method defines a structured training schedule for 3D Gaussian Splatting that combines LiDAR-guided initialization with diffusion-based pseudo-view guidance to improve reconstruction quality and convergence speed [S1]. It addresses the ambiguity and noise inherent in RGB-only initialization by leveraging depth priors and regularizing the optimization trajectory using generative priors [S1]. It builds on [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md) by introducing specific initialization and scheduling protocols.

## How it works
**Inputs:**
- LiDAR point clouds (sparse depth/geometry).
- RGB images (sparse views).
- A pre-trained Diffusion Model (e.g., Stable Diffusion) with a VAE encoder/decoder [S1].

**Outputs:**
- Optimized 3D Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients, color) [S1].

**Data Structures:**
- **3D Gaussian Model:** Standard 3DGS representation. Initialized with points derived from LiDAR [S1].
- **Pseudo Views:** Rendered images from random camera poses, used as input to the diffusion model [S1].

**Steps:**

1.  **LiDAR-to-Color Projection:**
    For each LiDAR frame, project the 3D LiDAR points onto the corresponding 2D image frame to assign RGB color values to each point [S1].

2.  **3D Reprojection:**
    Reproject the now-colored LiDAR points back into 3D space to create a set of colored 3D points [S1].

3.  **Voxel Downsampling:**
    Accumulate all colored point clouds from all frames. Perform voxel downsampling with a voxel size of 5 to reduce density and remove outliers [S1].

4.  **Warm-up Phase:**
    Train the 3D Gaussian model for 500 iterations using only the RGB images and LiDAR-initialized points. Do **not** sample pseudo views during this phase to allow the geometry to stabilize [S1].

5.  **Guided Phase:**
    For the remaining iterations, sample 4 pseudo views every 10 iterations [S1].
    - Render the current 3D Gaussian model from these random camera poses to obtain pseudo-view images [S1].
    - Pass the rendered pseudo-view through the VAE Encoder to obtain a latent feature map [S1].
    - Add noise at level $t$ to the latent feature, where $t \sim [t_{min}, t_{max}]$ [S1].
    - Denoise the latent feature using the Diffusion Model from level $t$ down to $t_{min}$ [S1].
    - Decode the denoised latent feature to obtain the generated guidance image [S1].
    - Use the generated image to compute a loss term that regularizes the training [S1].

6.  **Classifier-Free Guidance (Optional):**
    During the first training stage, set both text prompts and reference images to empty with a 10% probability to enable classifier-free guidance [S1].

**Parameters:**
- **Voxel Size:** 5 [S1].
- **Warm-up Iterations:** 500 [S1].
- **Pseudo-view Sampling Frequency:** 4 views every 10 iterations [S1].
- **Total Training Iterations:** 50,000 [S1].
- **Diffusion Noise Level:** $t_{max} = 10$ [S1]. $t_{min}$ is not explicitly given in the sources [S1].

**Edge Cases and Pitfalls:**
- **LiDAR Registration:** The quality of initialization depends on the accurate projection of LiDAR points to image frames. Misalignment will propagate errors [S1].
- **Diffusion Prior Strength:** The balance between the photometric loss and the diffusion guidance loss is critical. If the diffusion prior is too strong, it may hallucinate details not present in the input data [S1].
- **Noise Level Selection:** The choice of $t_{min}$ and $t_{max}$ affects the fidelity of the guidance. Higher noise levels provide more structural guidance but less detail [S1].

## Subtopics

### [Improved 3D Gaussian Splatting Optimization Pipeline](18760-improved-3d-gaussian-splatting-optimization-pipeline.md)
This method defines a structured training schedule for 3D Gaussian Splatting that combines multi-scale rendering, adaptive densification, and optimized spherical harmonics evaluation to improve reconstruction quality and convergence speed.

## Operating characteristics
- **Time Complexity:** The addition of diffusion model inference for pseudo views increases the per-iteration cost compared to standard 3DGS. However, the pseudo views are sampled sparsely (4 every 10 iterations), mitigating the overhead [S1].
- **Memory Complexity:** Requires storing the diffusion model and VAE in memory, in addition to the 3D Gaussian model [S1].
- **Performance:** Improves reconstruction quality, particularly in textureless or ambiguous regions, by leveraging LiDAR geometry and diffusion priors [S1].
- **Strengths:** Robust initialization, better handling of sparse views, improved geometry consistency [S1].
- **Limitations:** Requires LiDAR data for initialization, higher computational cost due to diffusion model [S1].

## Background
This method was introduced in "SGD: Street View Synthesis with Gaussian Splatting and Diffusion Prior" [S1]. It combines the strengths of 3D Gaussian Splatting for fast rendering with the generative capabilities of diffusion models for regularization. The use of LiDAR for initialization addresses the common issue of poor initialization in sparse-view scenarios [S1].

## Sources
[S1] SGD: Street View Synthesis with Gaussian Splatting and Diffusion Prior, https://doi.org/10.1109/wacv61041.2025.00375, fetched 2026-10-27
