# DiffSplat

## Overview
DiffSplat is a method that natively generates 3D Gaussian splats by fine-tuning a pretrained text-to-image diffusion model, exploiting web-scale 2D priors while maintaining 3D consistency [S1, unofficial]. It addresses the challenge of generating high-fidelity 3D geometry from text or single images by using a lightweight reconstruction model to bootstrap multi-view Gaussian-splat grids for dataset curation and a 3D rendering loss to enforce cross-view coherence [S1, unofficial]. It is distinguished from its sibling [HumanGaussian](7764-humangaussian.md) and parent [DreamGaussian](5998-dreamgaussian.md) by its native generation of Gaussian primitives via diffusion fine-tuning, achieving text- and image-to-3DGS in approximately 1-2 seconds [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A text prompt or a single reference image.
*   **Output:** A set of 3D Gaussian splats (positions, covariances, opacities, spherical harmonics coefficients) representing the 3D scene.

**Data Structures:**
The method utilizes a lightweight reconstruction model to produce multi-view Gaussian-splat grids. The specific internal layout of these grids is not detailed in the sources [S1, unofficial].

**Steps:**
1.  **Bootstrap Multi-View Grids:** Use a lightweight reconstruction model to generate initial multi-view Gaussian-splat grids. These grids serve as the basis for dataset curation and initial scene representation [S1, unofficial].
2.  **Fine-Tune Diffusion Model:** Fine-tune a pretrained text-to-image diffusion model. This step exploits web-scale 2D priors to guide the generation process [S1, unofficial].
3.  **Enforce 3D Consistency:** Apply a 3D rendering loss during the optimization process. This loss ensures that the generated Gaussians produce consistent views across different camera angles, maintaining 3D coherence [S1, unofficial].
4.  **Generate Splats:** Output the final set of 3D Gaussian splats optimized through the fine-tuning and loss enforcement steps [S1, unofficial].

**Parameters:**
*   **Inference Time:** ~1-2 seconds for text- and image-to-3DGS [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# DiffSplat Implementation Logic
# Note: Specific implementation details of the lightweight reconstruction model
# and the fine-tuning loop are not provided in the sources.

def diff_splat(text_prompt_or_image):
    # Step 1: Bootstrap multi-view Gaussian-splat grids
    # Uses a lightweight reconstruction model
    initial_grids = lightweight_reconstruction_model(text_prompt_or_image)
    
    # Step 2: Fine-tune pretrained text-to-image diffusion model
    # Exploits web-scale 2D priors
    diffusion_model = load_pretrained_diffusion_model()
    diffusion_model = fine_tune(diffusion_model, initial_grids)
    
    # Step 3: Enforce 3D consistency via 3D rendering loss
    # Optimizes the Gaussian splats to ensure cross-view coherence
    gaussians = optimize_gaussians(
        initial_grids, 
        diffusion_model, 
        loss_function="3d_rendering_loss"
    )
    
    return gaussians
```

## Subtopics

### LiDAR-based Gaussian Initialization
A method to initialize 3D Gaussians for a radiance field map using size-adaptive voxels derived from a LiDAR-inertial system, replacing traditional point clouds with ellipsoidal surface Gaussians to improve structural accuracy and handle non-Lambertian surfaces [S2]. It partitions the LiDAR point cloud into size-adaptive voxels, inspired by the octree approach in [12] [S2]. It estimates the surfaces of the scene using LiDAR measurements and develops these estimated surfaces into ellipsoidal surface Gaussians to serve as the initial map representation [S2]. The adaptiveness of the voxel partition is determined by evaluating a parameter $\eta$, which indicates whether a voxel has a surface with planar characteristics inside. Smaller voxels and further subdivision into finer levels are allowed to obtain a more precise map with normal vectors of surface Gaussians, allowing even curved surfaces to be approximated if sufficiently divided [S2]. This approach produces a photometrically accurate LiDAR-visual map, enhancing mapping precision and visual realism by optimizing Gaussians using photometric gradients [S2].

## Operating characteristics
*   **Time Complexity:** Inference time is approximately 1-2 seconds for text- and image-to-3DGS [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Natively generates 3D Gaussian splats, exploits web-scale 2D priors, maintains 3D consistency, and is fast (~1-2 seconds) [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** When fast, native generation of 3D Gaussian splats from text or single images is required, leveraging 2D diffusion priors [S1, unofficial].
*   **When not:** When high-fidelity mesh generation is required (use [DreamGaussian](5998-dreamgaussian.md) instead) or when dealing with LiDAR data (use LiDAR-based Gaussian Initialization) [S1, unofficial][S2].

## Background
DiffSplat is marked as a 2026 SOTA update [S1, unofficial]. It is part of the broader category of Generative 3D / image-to-3D with Gaussians, specifically within the Diffusion + SDS / SDS-variants branch [S1, unofficial]. It follows the lineage of methods like DreamGaussian, GaussianDreamer, LucidDreamer, HiFi-123, GS-Gen, and HumanGaussian, which perform text-to-3D via SDS over GS [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] LIV-GaussMap: LiDAR-Inertial-Visual Fusion for Real-Time 3D Radiance Field Map Rendering, https://doi.org/10.1109/lra.2024.3400149, fetched 2026-10-26
