# Curriculum Learning with Slice and Voxel Supervision

Also called: Slice and Voxel Supervision Curriculum

## Overview
This method stabilizes the optimization of 3D Gaussian representations for volumetric data by decoupling local structural learning from global spatial consistency. It addresses the instability caused by random voxel sampling and the lack of global guidance in slice-only supervision by enforcing a specific training order: planar slice constraints first, followed by sparse voxel supervision. This approach ensures the Gaussian cloud captures fine anatomical details without losing volumetric coherence, outperforming ablations that use random sampling or single-supervision modes [S1].

## How it works
The method optimizes a set of Gaussian kernels $\mathcal{G} = \{G_1, G_2, \dots, G_m\}$ initialized from voxel samples. Each Gaussian $G_i$ has learnable parameters: mean $\mu_i \in \mathbb{R}^3$, covariance $\Sigma_i$, opacity $\alpha_i$, and color $c_i$. The optimization follows a two-stage curriculum.

**Inputs:** A volumetric dataset (e.g., MRI, Cryosection) represented as a voxel grid; initial Gaussian set $\mathcal{G}$ centered at sampled voxel positions.
**Outputs:** Optimized Gaussian parameters $\mathcal{G}^*$.

**Data Structures:**
*   **Gaussian Set:** List of objects with fields `mean` (3 floats), `covariance` (3x3 matrix or 6 floats for symmetric), `opacity` (1 float), `color` (3 floats).
*   **Voxel Grid:** 3D array of intensity values.

**Steps:**

1.  **Stage 1: Slice Supervision (Structural Learning)**
    *   For a defined number of iterations (or until convergence on local structures):
        *   Sample a plane with arbitrary orientation.
        *   Select sparse pixels on this plane.
        *   Map each selected pixel to a 3D voxel coordinate.
        *   Render the projection of $\mathcal{G}$ onto this plane using shear-warp rendering principles.
        *   Compute loss against the ground-trax slice intensities.
        *   Update $\mathcal{G}$ parameters.
    *   *Rationale:* This stage incorporates finer structural details through planar constraints. It aligns with the shear-warp rendering process, which generates images from stacks of 2D slices [S1].

2.  **Stage 2: Voxel Supervision (Global Guidance)**
    *   After Stage 1, introduce voxel supervision.
    *   Sample sparse voxels from the volume.
    *   Compute loss against the voxel grid values at these coordinates.
    *   Update $\mathcal{G}$ parameters.
    *   *Rationale:* Sparse voxels provide global coverage. This stage establishes global spatial guidance, correcting the lack of volumetric consistency seen in slice-only training [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_gaussian_volumetric(voxel_grid, initial_gaussians, config):
    gaussians = initial_gaussians
    
    # Stage 1: Slice Supervision
    for iteration in range(config.slice_iterations):
        # Sample a plane with arbitrary orientation
        plane = sample_arbitrary_plane()
        
        # Select sparse pixels on the plane
        pixels = sample_sparse_pixels(plane, count=config.sparse_pixel_count)
        
        # Map pixels to 3D voxel coordinates
        voxel_coords = map_pixels_to_voxel_coords(pixels, plane)
        
        # Render slice projection
        rendered_slice = shear_warp_render(gaussians, plane)
        
        # Compute loss against ground truth slice
        # Note: Ground truth slice is extracted from voxel_grid using plane parameters
        gt_slice = extract_slice_from_voxel_grid(voxel_grid, plane)
        loss_slice = compute_loss(rendered_slice, gt_slice)
        
        # Update Gaussians
        gaussians = update_parameters(gaussians, loss_slice)

    # Stage 2: Voxel Supervision
    for iteration in range(config.voxel_iterations):
        # Sample sparse voxels
        voxel_indices = sample_sparse_voxels(voxel_grid, count=config.sparse_voxel_count)
        
        # Extract ground truth values
        gt_values = get_voxel_values(voxel_grid, voxel_indices)
        
        # Render/Query Gaussians at voxel positions
        # In Gaussian Splatting context, this often involves projecting to image space
        # or directly evaluating density at 3D points if using a differentiable volume renderer
        rendered_values = query_gaussian_values(gaussians, voxel_indices)
        
        loss_voxel = compute_loss(rendered_values, gt_values)
        
        gaussians = update_parameters(gaussians, loss_voxel)

    return gaussians
```

**Parameters:**
*   `slice_iterations`: Number of iterations for Stage 1. Not given in the sources.
*   `voxel_iterations`: Number of iterations for Stage 2. Not given in the sources.
*   `sparse_pixel_count`: Number of pixels sampled per plane. Not given in the sources.
*   `sparse_voxel_count`: Number of voxels sampled per iteration. Not given in the sources.

**Edge Cases:**
*   **Random Voxel Sampling:** Using random voxel sampling *before* or *during* slice supervision leads to unstable global approximation and destabilizes learning [S1].
*   **Slice-Only:** Using only slice supervision limits accurate reconstruction of the full volumetric structure due to lack of global spatial guidance [S1].

## Subtopics

### Curriculum Learning with Sparse Voxel and Slice Sampling
This variant explicitly defines the sampling strategy to align with shear-warp volume rendering. It samples a plane with arbitrary orientation, selects sparse pixels on the plane, and maps each pixel to a 3D voxel coordinate. This ensures the Gaussian representation learns spatial continuity across neighboring regions, capturing geometric structure and texture continuity while maintaining global coverage through sparse voxels [S1].

## Operating characteristics
*   **Performance:** Combining slice and voxel supervision yields higher PSNR and SSIM than using only slice supervision (A4) or random voxel sampling (A2, A3) [S1].
*   **Limitations of Alternatives:**
    *   Slice-only (A4) lacks global spatial guidance [S1].
    *   Random voxel then random slices (A2) leads to unstable global approximation [S1].
    *   Random slices then random voxel (A3) results in unstable learning of volumetric structure because global constraints are not established early enough [S1].
    *   MCE voxel-based supervision (A5) stabilizes optimization but lacks structured planar constraints, leading to lower performance in rendered views [S1].
*   **Strengths:** Efficiently learns both global volumetric consistency and local anatomical details. Aligns naturally with shear-warp rendering [S1].

## Background
The method is part of a framework for Gaussian Volumetric Representation designed for efficient Shear-Warp Visualization. It is applicable to multimodal volumetric datasets including MRI and Cryosection volumes. The approach leverages curriculum learning to progressively train sparse voxel sampling and structured slice-based sampling [S1].

## Sources
[S1] Gaussian Volumetric Representation for Efficient Shear-Warp Visualization, https://doi.org/10.1007/978-3-032-37461-5_35, fetched 2026-10-26
