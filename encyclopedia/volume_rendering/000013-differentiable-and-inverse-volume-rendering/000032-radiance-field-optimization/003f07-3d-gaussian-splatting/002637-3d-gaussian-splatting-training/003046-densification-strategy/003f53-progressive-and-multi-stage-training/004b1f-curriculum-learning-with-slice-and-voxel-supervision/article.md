# Curriculum Learning with Slice and Voxel Supervision

Also called: Slice and Voxel Supervision Curriculum

## Overview
This method addresses the instability and incomplete volumetric reconstruction inherent in optimizing 3D Gaussian representations for shear-warp rendering by decoupling global spatial structure learning from local geometric detail acquisition. It sets itself apart from sibling methods like [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md) by explicitly leveraging the structural alignment between 2D slice-based supervision and the shear-warp rendering pipeline, rather than relying solely on resolution scaling or densification heuristics.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D image slices (or projections) from a volumetric dataset (e.g., MRI, Cryosection), and an initial set of $m$ Gaussian kernels $\mathcal{G} = \{G_1, G_2, \dots, G_m\}$, where each $G_i$ is initialized at a sampled voxel position with learnable mean $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and color $c_i$.
*   **Output:** Optimized Gaussian parameters $\mathcal{G}$ that reconstruct the volume with high fidelity under shear-warp rendering.

**Data Structures:**
*   **Gaussian Kernel:** Defined by mean $\mu \in \mathbb{R}^3$, covariance $\Sigma \in \mathbb{R}^{3 \times 3}$ (or 6 independent parameters for diagonal/axis-aligned), opacity $\alpha \in [0,1]$, and spherical harmonics coefficients for color $c$.
*   **Voxel Grid:** A 3D grid where sparse voxels provide global coverage.

**Numbered Steps:**
1.  **Initialization:** Initialize one Gaussian kernel at each of $m$ sampled voxel positions.
2.  **Stage 1: Slice Supervision:**
    *   In each training iteration, sample a plane with arbitrary orientation.
    *   Select sparse pixels on the sampled plane.
    *   Map each selected pixel to a 3D voxel coordinate.
    *   Render the image using the current Gaussian set via shear-warp rendering (stacking 2D slices).
    *   Compute loss between rendered slice and ground-truth slice.
    *   Update Gaussian parameters to minimize this loss. This stage progressively incorporates finer structural details through planar constraints.
3.  **Stage 2: Voxel Supervision:**
    *   After establishing initial structural consistency, introduce voxel supervision.
    *   Perform random sampling of voxels with dense approximation.
    *   Compute loss based on the Gaussian values at these voxel positions against ground-truth voxel data.
    *   Update Gaussian parameters. This establishes global spatial guidance.
4.  **Combined Optimization:** Continue optimizing with both slice and voxel supervision to balance local continuity and global consistency.

**Formulas:**
*   **Shear-Warp Rendering:** The volume is decomposed into a stack of 2D slices. For a given view, the volume is sheared, and slices are projected and accumulated. The color $C$ at a pixel is computed as:
    $$ C = \sum_{i=1}^{N} T_i \alpha_i c_i $$
    where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the accumulated transmittance, and $N$ is the number of Gaussians along the ray.
*   **Loss Function:** The total loss $\mathcal{L}$ is a weighted sum of slice loss $\mathcal{L}_{slice}$ and voxel loss $\mathcal{L}_{voxel}$:
    $$ \mathcal{L} = \lambda_{slice} \mathcal{L}_{slice} + \lambda_{voxel} \mathcal{L}_{voxel} $$
    where $\mathcal{L}_{slice}$ is typically an $L_1$ or $L_2$ loss between rendered and ground-truth slice pixels, and $\mathcal{L}_{voxel}$ is the loss between Gaussian values and ground-truth voxel intensities.

**Parameters:**
*   $\lambda_{slice}, \lambda_{voxel}$: Weights for slice and voxel losses. Typical values are not specified in the sources, but curriculum learning implies $\lambda_{slice}$ dominates early stages.
*   $m$: Number of initial Gaussians. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Random Voxel Sampling:** Using random voxel sampling without dense approximation leads to unstable global approximation and destabilizes Gaussian parameter learning [S1].
*   **Slice-First without Voxel:** Applying slice supervision first without subsequent voxel supervision (A4) limits accurate reconstruction of the full volumetric structure due to lack of global spatial guidance [S1].
*   **Voxel-First with Random Slices:** Applying slice supervision after random voxel sampling (A3) results in unstable learning because slice supervision is introduced before stable global voxel constraints are established [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_gaussian_volume(dataset, initial_gaussians, num_iterations):
    gaussians = initial_gaussians
    # Stage 1: Slice Supervision
    for iteration in range(num_iterations // 2):
        # Sample a plane with arbitrary orientation
        plane = sample_arbitrary_plane()
        # Select sparse pixels on the plane
        pixels = select_sparse_pixels(plane)
        # Map pixels to 3D voxel coordinates
        voxel_coords = map_pixels_to_voxels(pixels, plane)
        # Render using shear-warp
        rendered_slice = shear_warp_render(gaussians, plane)
        # Compute loss against ground-truth slice
        loss_slice = compute_loss(rendered_slice, get_ground_truth_slice(plane))
        # Update Gaussians
        update_gaussians(gaussians, loss_slice)
        
    # Stage 2: Voxel Supervision (with dense approximation)
    for iteration in range(num_iterations // 2, num_iterations):
        # Randomly sample voxels with dense approximation
        voxel_samples = random_sample_voxels_dense()
        # Render or query Gaussian values at voxel positions
        gaussian_values = query_gaussians_at_voxels(gaussians, voxel_samples)
        # Compute loss against ground-truth voxel data
        loss_voxel = compute_loss(gaussian_values, get_ground_truth_voxels(voxel_samples))
        # Update Gaussians
        update_gaussians(gaussians, loss_voxel)
        
    return gaussians
```

## Subtopics

### Curriculum Learning with Sparse Voxel and Slice Sampling
This variant explicitly aligns the sampling strategy with shear-warp volume rendering. Sparse voxels provide global coverage of the volume, while slice samples capture spatially correlated regions that encode geometric structure and texture continuity. In each training iteration, a plane with arbitrary orientation is sampled, sparse pixels are selected on the plane, and each pixel is mapped to a 3D voxel coordinate. This allows the Gaussian representation to learn spatial continuity across neighboring regions, naturally aligning the learned representation with the rendering process [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Combining slice and voxel supervision yields higher PSNR and SSIM compared to using only slice supervision (A4) or random voxel sampling [S1].
*   **Strengths:** Efficiently learns both global volumetric consistency and local anatomical details. Aligns with shear-warp rendering structure.
*   **Limitations:** Requires careful scheduling of slice vs. voxel supervision. Random voxel sampling without dense approximation destabilizes learning [S1].
*   **When to Use:** When reconstructing volumetric data (e.g., medical images) where global structure and local texture are both critical, and shear-warp rendering is the target.
*   **When Not to Use:** When real-time performance is not required and simpler volumetric representations suffice, or when the rendering pipeline does not support slice-based decomposition.

## Background
Originated in the context of Gaussian Volumetric Representation for Efficient Shear-Warp Visualization [S1]. The method leverages curriculum learning [37] to progressively train Gaussian representations using sparse voxel sampling and structured slice-based sampling. It is applicable to multimodal volumetric datasets including MRI and Cryosection volumes, enabling efficient reconstruction and real-time selective visualization of organ structures [S1].

## Sources
[S1] Gaussian Volumetric Representation for Efficient Shear-Warp Visualization, https://doi.org/10.1007/978-3-032-37461-5_35, fetched 2026-10-26
