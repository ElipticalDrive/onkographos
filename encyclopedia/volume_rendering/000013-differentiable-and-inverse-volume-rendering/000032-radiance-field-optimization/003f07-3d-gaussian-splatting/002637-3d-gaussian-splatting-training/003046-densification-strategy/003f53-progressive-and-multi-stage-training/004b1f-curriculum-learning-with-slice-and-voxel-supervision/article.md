# Curriculum Learning with Slice and Voxel Supervision

Also called: Slice and Voxel Supervision Curriculum

## Overview
This method addresses the instability and local minima traps in 3D Gaussian Splatting training by decoupling global volumetric consistency from local structural detail learning. It sets itself apart from siblings like [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md) by explicitly alternating between sparse voxel supervision for global geometry and structured slice supervision for local texture, rather than relying solely on pixel-level photometric loss or random sampling schedules.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D images (views) of a volumetric scene (e.g., MRI, Cryosection), and an initial set of 3D Gaussian kernels $\mathcal{G} = \{G_1, G_2, \dots, G_m\}$ initialized at sampled voxel positions [S1].
*   **Output:** Optimized Gaussian parameters (mean $\mu$, covariance $\Sigma$, opacity $\alpha$, color $c$) that reconstruct the volume.

**Data Structures:**
*   **Gaussian Set $\mathcal{G}$:** Each Gaussian $G_i$ has learnable parameters: mean $\mu \in \mathbb{R}^3$, covariance $\Sigma \in \mathbb{R}^{3 \times 3}$ (or 6 independent values), opacity $\alpha \in [0,1]$, and spherical harmonics coefficients for color [S1].
*   **Voxel Grid:** A 3D grid representing the volume, where each voxel can be mapped to a Gaussian center [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize $m$ Gaussian kernels centered at sampled voxel positions from the input volume data [S1].
2.  **Stage 1: Slice Supervision:**
    *   In each training iteration, sample a plane with arbitrary orientation [S1].
    *   Select sparse pixels on this sampled plane [S1].
    *   Map each selected pixel to a 3D voxel coordinate [S1].
    *   Render the view through these slices and compute loss against ground truth slice images. This incorporates finer structural details through planar constraints [S1].
3.  **Stage 2: Voxel Supervision:**
    *   After establishing initial structure, introduce voxel supervision under random sampling with dense approximation [S1].
    *   Sample sparse voxels globally to provide spatial guidance [S1].
    *   Compute loss based on voxel values to stabilize global approximation [S1].
4.  **Optimization:** Update Gaussian parameters using the combined or staged loss functions. The curriculum ensures that global spatial guidance is established before or concurrently with local planar constraints to prevent instability [S1].

**Formulas:**
*   **Gaussian Kernel:** $G_i(\mathbf{x}) = \exp\left(-\frac{1}{2}(\mathbf{x}-\mu_i)^T \Sigma_i^{-1} (\mathbf{x}-\mu_i)\right)$ [S1].
*   **Shear-Warp Rendering:** The rendering process generates images from stacks of 2D slices extracted from the 3D volume, aligning the learned representation with this process [S1].

**Parameters:**
*   **Sampling Strategy:** Sparse voxel sampling and structured slice-based sampling [S1].
*   **Curriculum Order:** Slice supervision first, followed by voxel supervision with dense approximation [S1].

**Edge Cases and Pitfalls:**
*   **Instability with Random Voxels First:** Applying random voxel sampling before stable global constraints are established leads to unstable learning of Gaussian parameters [S1].
*   **Lack of Global Guidance with Slices Only:** Using only slice supervision limits accurate reconstruction of the full volumetric structure due to lack of global spatial guidance [S1].
*   **Precision:** Mapping 2D slice pixels to 3D voxel coordinates requires precise alignment to ensure spatial continuity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_gaussian_volumetric(scene_data, initial_gaussians, num_iterations):
    """
    Train Gaussian representation using curriculum learning with slice and voxel supervision.
    """
    gaussians = initial_gaussians  # Set of G_i with mean, covariance, opacity, color
    
    # Stage 1: Slice Supervision
    for iteration in range(num_iterations // 2):
        # Sample a plane with arbitrary orientation
        plane = sample_arbitrary_plane()
        
        # Select sparse pixels on the plane
        slice_pixels = select_sparse_pixels(plane)
        
        # Map pixels to 3D voxel coordinates
        voxel_coords = map_pixels_to_voxels(slice_pixels, plane)
        
        # Render slice view
        rendered_slice = render_slice(gaussians, plane)
        
        # Compute loss against ground truth slice
        loss_slice = compute_loss(rendered_slice, get_ground_truth_slice(scene_data, plane))
        
        # Update Gaussians
        update_gaussians(gaussians, loss_slice)

    # Stage 2: Voxel Supervision (with dense approximation)
    for iteration in range(num_iterations // 2, num_iterations):
        # Sample sparse voxels globally
        sampled_voxels = sample_sparse_voxels(scene_data)
        
        # Render voxel-based view or compute voxel loss
        rendered_voxels = render_voxels(gaussians, sampled_voxels)
        
        # Compute loss against ground truth voxels
        loss_voxel = compute_loss(rendered_voxels, get_ground_truth_voxels(scene_data, sampled_voxels))
        
        # Update Gaussians
        update_gaussians(gaussians, loss_voxel)
        
    return gaussians
```

## Subtopics

### Curriculum Learning with Sparse Voxel and Slice Sampling
This variant explicitly aligns the sampling strategy with shear-warp volume rendering. It uses sparse voxels for global coverage and slice samples for spatially correlated regions encoding geometric structure and texture continuity. In each iteration, a plane with arbitrary orientation is sampled, sparse pixels are selected, and each pixel is mapped to a 3D voxel coordinate, allowing the Gaussian representation to learn spatial continuity across neighboring regions [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians and the cost of shear-warp rendering. Slice sampling may be faster per iteration than full volume rendering but requires careful plane sampling [S1].
*   **Memory Complexity:** Proportional to the number of Gaussians and the voxel grid size if stored explicitly [S1].
*   **Performance:** Combining both slice and voxel supervision yields higher PSNR/SSIM than using only slice supervision (A4) or random voxel sampling (A2, A3) [S1].
*   **Strengths:** Efficiently learns both global volumetric consistency and local anatomical details. Aligns naturally with shear-warp rendering [S1].
*   **Limitations:** Requires careful curriculum scheduling; incorrect order (e.g., random voxels first) leads to instability [S1].

## Background
Originated in the context of Gaussian Volumetric Representation for Efficient Shear-Warp Visualization, this method was developed to optimize Gaussian representations for multimodal volumetric datasets like MRI and Cryosection volumes. It leverages curriculum learning to progressively train sparse voxel and slice-based sampling strategies [S1].

## Sources
[S1] Gaussian Volumetric Representation for Efficient Shear-Warp Visualization, https://doi.org/10.1007/978-3-032-37461-5_35, fetched 2026-10-26
