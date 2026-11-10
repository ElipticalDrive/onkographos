# DreamGaussian

Also called: DreamGaussian: Generative 3D Gaussian Splatting

## Overview
DreamGaussian is a text-to-3D generation method that replaces the MipNeRF representation in the DreamFusion framework with 3D Gaussian Splatting (3DGS), using Score Distillation Sampling (SDS) loss to optimize 3D Gaussians [S1]. It is distinguished from its siblings by its two-stage pipeline: initial SDS optimization of Gaussians followed by UV-texture refinement on an extracted mesh, enabling the generation of a textured mesh in approximately 2 minutes [S2, unofficial]. Unlike [Gaussian Asset Generation](7625-gaussian-asset-generation.md) which may use feed-forward models, DreamGaussian relies on iterative optimization [S2, unofficial].

## How it works
**Inputs:** A text prompt [S1].
**Outputs:** A textured 3D mesh [S2, unofficial].

**Data Structures:**
The method utilizes the standard 3D Gaussian Splatting representation [S1]. Each Gaussian $i$ is defined by:
- Mean position $\mu_i \in \mathbb{R}^3$
- Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (parameterized as a rotation matrix $R_i \in SO(3)$ and a scaling vector $s_i \in \mathbb{R}^3$) [G]
- Opacity $\alpha_i \in [0, 1]$ [G]
- Spherical Harmonics coefficients for view-dependent color $c_i$ [G]

**Steps:**
1. **Initialization:** Initialize a set of 3D Gaussians. The specific initialization strategy is not given in the sources [S1]. [G] Typically, this involves random sampling or uniform distribution within a bounding box.
2. **SDS Optimization:** Optimize the Gaussian parameters (positions, covariances, opacities, colors) using the SDS loss [S1].
   - Render images from random camera viewpoints [G].
   - Compute the SDS gradient using a pretrained 2D diffusion model [G].
   - Update Gaussians using gradient descent [G].
   - Apply the 3DGS splitting process during optimization to densify the representation where needed [S1].
3. **Mesh Extraction:** Extract a mesh from the optimized 3D Gaussians [S1]. This is typically done using a NeuS-like method to convert Gaussian density/opacity into a Signed Distance Field (SDF) [S1].
4. **UV Texture Refinement:** Refine the texture details of the extracted mesh [S1].
   - Map the mesh to UV coordinates [G].
   - Optimize UV texture colors using a pixel-wise Mean Squared Error (MSE) loss [S1].

**Parameters:**
- Optimization steps for SDS: Not given in the sources.
- Mesh extraction resolution: Not given in the sources.
- UV texture resolution: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Janus Problem:** Multi-face artifacts may occur. GSGEN introduces a 3D SDS loss to mitigate this [S1].
- **Oversaturation:** Addressed in variants like HumanGaussian using negative prompt guidance [S1].

## Subtopics

### DiffSplat
A method that natively generates 3D Gaussian splats by fine-tuning a pretrained text-to-image diffusion model, exploiting web-scale 2D priors while keeping 3D consistency [S2, unofficial]. It uses a lightweight reconstruction model to bootstrap multi-view Gaussian-splat grids for dataset curation and a 3D rendering loss for cross-view coherence [S2, unofficial]. Inference time is ~1-2 seconds for text- and image-to-3DGS [S2, unofficial]. Marked as a 2026 SOTA update [S2, unofficial].

### Generative Gaussian Splatting
A method that generates unbounded 3D cities by progressively extending the Gaussian cloud outward from a given seed region with a diffusion prior [S3, unofficial].

### HumanGaussian
A variant that addresses the oversaturation issue using negative prompt guidance [S1]. It initializes 3D Gaussians by randomly sampling points on the surface of the SMPL-X template [S1]. It extends Stable Diffusion to generate RGB and depth simultaneously [S1]. A dual-branch SDS is constructed as the optimization guidance [S1]. The classifier score from the null text prompt and the negative score from the negative prompt are combined to construct negative prompt guidance [S1].

## Operating characteristics
- **Time Complexity:** The method involves iterative optimization, which is generally slower than feed-forward methods. However, 3DGS rendering is high-FPS and low-memory compared to NeRF [S1].
- **Memory Complexity:** 3DGS representation has small memory footprint compared to NeRF [S1].
- **Performance:** Generates a textured mesh in 2 minutes [S2, unofficial].
- **Strengths:** High-resolution rendering, high FPS, small memory usage, suitable for generative settings under SDS loss due to the splitting process [S1].
- **Limitations:** Optimization time can be increased compared to feed-forward methods [S1].

## Background
DreamGaussian was introduced by Tang et al. in ICLR'24 [S2, unofficial]. It replaces the MipNeRF representation in the DreamFusion framework with 3DGS [S1]. It follows the idea of Magic3D by extracting a mesh and refining texture via UV optimization [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
