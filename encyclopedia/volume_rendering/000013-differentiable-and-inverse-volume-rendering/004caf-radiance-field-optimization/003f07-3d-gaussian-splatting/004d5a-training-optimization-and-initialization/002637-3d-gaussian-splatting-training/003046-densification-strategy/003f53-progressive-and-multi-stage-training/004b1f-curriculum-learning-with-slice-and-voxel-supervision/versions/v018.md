# Curriculum Learning with Slice and Voxel Supervision

Also called: Slice and Voxel Supervision Curriculum

## Overview
This method addresses the instability and local minima traps in 3D Gaussian Splatting training by decoupling the optimization of global volumetric structure from local planar details. It sets itself apart from sibling methods like [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md) by explicitly alternating between sparse voxel supervision for global consistency and structured slice supervision for local geometric continuity, leveraging the alignment with shear-warp rendering pipelines.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D images (or slices) from a volumetric dataset (e.g., MRI, Cryosection); initial 3D Gaussian parameters $\mathcal{G} = \{\mathbf{G}_1, \dots, \mathbf{G}_m\}$ initialized at sampled voxel positions.
*   **Output:** Optimized Gaussian parameters (means $\boldsymbol{\mu}$, covariances $\boldsymbol{\Sigma}$, opacities $\alpha$, colors $\mathbf{c}$) representing the volume.

**Data Structures:**
*   **Gaussian Set $\mathcal{G}$:** A list of $m$ Gaussians. Each $\mathbf{G}_i$ contains:
    *   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Mean position.
    *   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (or quaternion/scale representation).
    *   $\alpha_i \in [0, 1]$: Opacity.
    *   $\mathbf{c}_i \in \mathbb{R}^3$: Color (SH coefficients if applicable).
*   **Voxel Grid:** A 3D grid where sparse voxels are sampled.
*   **Slice Plane:** Defined by a normal vector and distance from origin, or by camera pose in shear-warp context.

**Steps:**
1.  **Initialization:** Initialize $m$ Gaussian kernels centered at randomly sampled voxel positions from the input volume [S1].
2.  **Curriculum Phase 1 (Slice Supervision):**
    *   Sample a plane with arbitrary orientation [S1].
    *   Select sparse pixels on this plane [S1].
    *   Map each selected pixel to a 3D voxel coordinate [S1].
    *   Render the image using the current Gaussian set via shear-warp or standard differentiable volume rendering.
    *   Compute loss against the ground-truth slice/image at these pixel locations.
    *   Update Gaussian parameters to minimize this loss. This phase incorporates finer structural details through planar constraints [S1].
3.  **Curriculum Phase 2 (Voxel Supervision):**
    *   Sample sparse voxels from the 3D volume [S1].
    *   Render the volume (or specific views) to establish global spatial guidance [S1].
    *   Compute loss against the ground-truth voxel values or rendered views.
    *   Update Gaussian parameters. This phase uses dense approximation under random sampling to stabilize global structure [S1].
4.  **Iteration:** Repeat steps 2 and 3, potentially alternating or combining them, until convergence. The curriculum ensures that global constraints are not destabilized by local slice updates and vice versa [S1].

**Formulas:**
*   **Gaussian Kernel:** $G_i(\mathbf{x}) = \exp\left(-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu}_i)^T \boldsymbol{\Sigma}_i^{-1} (\mathbf{x}-\boldsymbol{\mu}_i)\right)$ [S1].
*   **Rendering:** Composites Gaussians along rays or shear planes. For shear-warp, the volume is sheared, sliced, and projected [S1].
*   **Loss:** Typically MSE or L1 between rendered pixels/voxels and ground truth.

**Parameters:**
*   **Sampling Density:** Sparse for voxels, sparse for slice pixels. Exact numbers not given in sources [S1].
*   **Curriculum Schedule:** Not explicitly defined, but implies a sequence: Slice first, then Voxel, or alternating [S1].

**Edge Cases and Pitfalls:**
*   **Random Voxel First:** Applying random voxel supervision before stable slice constraints leads to unstable global approximation and destabilizes learning [S1].
*   **Slice Only:** Using only slice supervision limits accurate reconstruction of the full volumetric structure due to lack of global spatial guidance [S1].
*   **Random Slice First:** Applying slice supervision before stable global voxel constraints results in unstable learning of volumetric structure, which later random voxels cannot sufficiently correct [S1].

## Subtopics

### Curriculum Learning with Sparse Voxel and Slice Sampling
This variant explicitly defines the sampling strategy. Sparse voxels provide global coverage, while slice samples capture spatially correlated regions encoding geometric structure and texture continuity [S1]. It aligns with shear-warp rendering, which generates images from stacks of 2D slices [S1]. In each iteration, a plane with arbitrary orientation is sampled, sparse pixels are selected, and each pixel is mapped to a 3D voxel coordinate [S1]. This allows the Gaussian representation to learn spatial continuity across neighboring regions [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $m$ and the number of samples per iteration. Shear-warp rendering is generally efficient $O(N)$ where $N$ is the number of voxels/samples [S1].
*   **Memory Complexity:** $O(m)$ for Gaussian parameters.
*   **Performance:** Combining both slice and voxel supervision yields higher PSNR/SSIM than using only slice supervision (A4) or random voxel sampling (A2, A3) [S1].
*   **Strengths:** Efficiently learns both global volumetric consistency and local anatomical details [S1]. Stabilizes optimization by separating global and local constraints [S1].
*   **Limitations:** Requires careful scheduling of the curriculum. Random voxel sampling without structure can destabilize learning [S1].
*   **When to Use:** Multimodal volumetric datasets (MRI, Cryosection) where both global structure and local texture are important [S1].
*   **When Not to Use:** When computational resources are extremely limited and only global structure is needed (MCE voxel-based supervision might be preferred for stability without slice complexity, though it shows lower performance in rendered views) [S1].

## Background
Originated in the context of Gaussian Volumetric Representation for Efficient Shear-Warp Visualization [S1]. It builds on the idea that shear-warp rendering operates on stacks of 2D slices, so incorporating slice-based supervision aligns the learned representation with the rendering process [S1]. The method is applicable to multimodal volumetric datasets including MRI and Cryosection volumes [S1].

## Sources
[S1] Gaussian Volumetric Representation for Efficient Shear-Warp Visualization, https://doi.org/10.1007/978-3-032-37461-5_35, fetched 2026-10-26
