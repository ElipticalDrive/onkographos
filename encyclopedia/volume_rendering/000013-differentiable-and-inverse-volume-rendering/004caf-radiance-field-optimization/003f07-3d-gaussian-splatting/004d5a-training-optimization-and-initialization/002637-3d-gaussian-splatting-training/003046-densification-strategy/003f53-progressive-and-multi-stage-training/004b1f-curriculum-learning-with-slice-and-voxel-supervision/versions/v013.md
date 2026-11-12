# Curriculum Learning with Slice and Voxel Supervision

Also called: Slice and Voxel Supervision Curriculum

## Overview
This method structures the training of 3D Gaussian representations for volumetric data by progressively introducing supervision signals to balance local geometric detail with global spatial consistency. It solves the instability caused by random voxel sampling and the lack of global guidance in slice-only supervision [S1]. It sets itself apart from siblings like [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md) by explicitly leveraging the alignment between the learning process and the shear-warp rendering pipeline, using planar constraints before volumetric ones [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D volumetric data (e.g., MRI, Cryosection) and a target camera view.
*   **Output:** A set of 3D Gaussians $\mathcal{G} = \{G_1, \dots, G_m\}$ optimized to represent the volume.

**Data Structures:**
*   **Gaussian Set $\mathcal{G}$:** A collection of $m$ Gaussian kernels.
*   **Gaussian Parameters $G_i$:** Each Gaussian $i$ has a learnable mean $\mu_i \in \mathbb{R}^3$, covariance $\Sigma_i$, opacity $\alpha_i$, and color $c_i$ [S1].
*   **Voxel Grid:** The underlying volumetric space from which samples are drawn.

**Steps:**
1.  **Initialization:** Initialize one Gaussian kernel centered at each sampled voxel position from the input volume [S1].
2.  **Stage 1: Slice Supervision:**
    *   Sample a plane with arbitrary orientation [S1].
    *   Select sparse pixels on this sampled plane [S1].
    *   Map each selected pixel to a 3D voxel coordinate [S1].
    *   Render the view using the current Gaussian set and compute the loss against the ground truth image at these pixel locations.
    *   Optimize Gaussian parameters using this loss. This stage establishes local geometric structure and texture continuity [S1].
3.  **Stage 2: Voxel Supervision:**
    *   Introduce voxel supervision under random sampling with dense approximation [S1].
    *   Sample voxels from the volume.
    *   Compute the loss based on the Gaussian representation at these voxel positions against the ground truth volume data.
    *   Optimize Gaussian parameters. This stage provides global spatial guidance, correcting the limited coverage of slice-only supervision [S1].

**Formulas:**
*   **Gaussian Kernel:** $G_i(\mathbf{x}) = \alpha_i \exp\left(-\frac{1}{2}(\mathbf{x}-\mu_i)^T \Sigma_i^{-1} (\mathbf{x}-\mu_i)\right) c_i$ [S1].
*   **Loss Function:** The total loss combines the rendering loss from slice supervision and the voxel reconstruction loss from voxel supervision. The specific weighting is not given in the sources [S1].

**Parameters:**
*   **Sampling Strategy:** Sparse pixels for slices; random dense approximation for voxels [S1].
*   **Order:** Slice supervision first, then voxel supervision [S1].

**Edge Cases and Pitfalls:**
*   **Slice-only (A4):** Using only slice supervision limits accurate reconstruction of the full volumetric structure due to lack of global spatial guidance [S1].
*   **Random Voxel First (A3):** Applying slice supervision before stable global voxel constraints are established leads to unstable learning of the volumetric structure, as the initial optimization relies on limited planar information [S1].
*   **Random Voxel + Random Slice (A2):** Introducing slice supervision together with voxel supervision under random sampling leads to unstable global approximation and destabilizes the learning of Gaussian parameters [S1].

## Subtopics

### Curriculum Learning with Sparse Voxel and Slice Sampling
This variant explicitly defines the sampling strategy to align with shear-warp volume rendering. Sparse voxels provide global coverage of the volume, while slice samples capture spatially correlated regions that encode geometric structure and texture continuity [S1]. In each training iteration, a plane with arbitrary orientation is sampled, sparse pixels are selected on the plane, and each pixel is mapped to a 3D voxel coordinate [S1]. This allows the Gaussian representation to learn spatial continuity across neighboring regions [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Combining both slice and voxel supervision yields higher PSNR/SSIM than using only slice supervision [S1].
*   **Strengths:** Efficiently learns both global volumetric consistency and local anatomical details [S1]. Aligns naturally with the shear-warp rendering process [S1].
*   **Limitations:** Random voxel sampling alone leads to unstable global approximation [S1]. Slice-only supervision lacks global spatial guidance [S1].
*   **When to use:** For multimodal volumetric datasets (MRI, Cryosection) where both global structure and local texture are important [S1].
*   **When not to use:** When only local planar details are needed, or when computational resources are extremely limited and global consistency is less critical than speed [S1].

## Background
This method was proposed in the context of Gaussian Volumetric Representation for Efficient Shear-Warp Visualization [S1]. It builds on the observation that shear-warp rendering operates on stacks of 2D slices, making slice-based supervision a natural fit for training Gaussian representations intended for this rendering pipeline [S1]. The approach is applicable to multimodal volumetric datasets including MRI and Cryosection volumes [S1].

## Sources
[S1] Gaussian Volumetric Representation for Efficient Shear-Warp Visualization, https://doi.org/10.1007/978-3-032-37461-5_35, fetched 2026-10-26
