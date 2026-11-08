# Training Pipeline

Also called: Iterative optimization loop

## Overview
The training pipeline for 3D Gaussian Splatting is an iterative optimization process that minimizes a differentiable loss function between rendered images and ground-truth views by updating Gaussian parameters [S1, S2, unofficial]. It distinguishes itself from standard neural field optimization by using a differentiable rasterizer to compute gradients directly on explicit geometric primitives (Gaussians) rather than implicit neural networks, allowing for adaptive control (pruning, cloning, splitting) of the scene representation during training [S1, S2, unofficial].

## How it works
**Inputs:**
*   Sparse point cloud and extra features (e.g., color, SH coefficients) [S1, unofficial].
*   Camera poses for the current batch of views [S1, unofficial].
*   Ground-truth images from multiple views [S1, unofficial].

**Outputs:**
*   Updated Gaussian parameters (position, covariance, opacity, color, SH coefficients) [S1, S2, unofficial].
*   Rasterized images for loss computation [S1, S2, unofficial].

**Data Structures:**
*   `WeightToTrain`: State container holding the current set of Gaussians (`sparsePointCloud`) and their attributes (`pointCloudExtraFeatures`) [S1, unofficial].
*   `Rasterizer`: Differentiable module that maps 3D Gaussians to 2D pixel colors [S1, unofficial].

**Algorithm Steps:**
1.  **Rasterization:** The `Rasterizer` takes the current `sparsePointCloud` and `cameraPose` as input. It projects the 3D Gaussians onto the 2D image plane, sorts them by depth, and alpha-blends their contributions to produce the rasterized image $\mathbf{I}_r$ [S1, S2, unofficial].
2.  **Loss Computation:** The rasterized image $\mathbf{I}_r$ is compared against the ground-truth image $\mathbf{I}_{gt}$ (or $\mathbf{I}_{\text{tet}}$ in CT contexts) using a combination of L1 and Structural Similarity (SSIM) losses [S2, unofficial].
    *   For FaCT-GS, a 3D total variation loss $\mathcal{L}_{TV}$ is also computed on a randomly sampled voxelized volume section $\mathbf{V}_{32^3}$ to enforce homogeneity [S2, unofficial].
    *   The total loss $\mathcal{C}_{\text{total}}$ is calculated as:
        $$ \mathcal{C}_{\text{total}} = \mathcal{L}_1 (\mathbf{I}_r, \mathbf{I}_{gt}) + \alpha_{\text{SSIM}} \mathcal{L}_{\text{SSIM}} (\mathbf{I}_r, \mathbf{I}_{gt}) + \alpha_{\text{TV}} \mathcal{L}_{\text{TV}} (\mathbf{V}_{32^3}) $$
        where $\alpha_{\text{SSIM}}$ and $\alpha_{\text{TV}}$ are weighting hyperparameters [S2, unofficial].
3.  **Backpropagation:** Gradients $\frac{\partial \mathcal{C}_{\text{total}}}{\partial \theta}$ are computed with respect to all Gaussian parameters $\theta$ (position, covariance, opacity, color, SH coefficients) via the differentiable rasterizer [S1, S2, unofficial].
4.  **Parameter Update:** The optimizer updates the parameters in `WeightToTrain` using the computed gradients [S1, unofficial].
5.  **Adaptive Control (FaCT-GS specific):** In the backward pass or subsequent steps, the loss gradients are used to perform adaptive control operations: pruning low-opacity Gaussians, cloning high-gradient Gaussians, or splitting Gaussians to increase density where needed [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def training_pipeline_iteration(gaussians, camera_poses, gt_images, fact_gs_mode=False):
    """
    One iteration of the 3D Gaussian Splatting training loop.
    
    Args:
        gaussians: Current state of Gaussians (position, covariance, opacity, color, SH).
        camera_poses: Camera extrinsics and intrinsics for the current batch.
        gt_images: Ground truth images corresponding to camera_poses.
        fact_gs_mode: Boolean flag to enable FaCT-GS specific regularization and adaptive control.
        
    Returns:
        Updated gaussians state.
    """
    # 1. Rasterization
    # Rasterizer projects 3D Gaussians to 2D and alpha-blends to produce rendered image
    rendered_image = rasterizer(gaussians, camera_poses)
    
    # 2. Loss Computation
    # Standard L1 and SSIM losses
    l1_loss = compute_l1_loss(rendered_image, gt_images)
    ssim_loss = compute_ssim_loss(rendered_image, gt_images)
    
    total_loss = l1_loss + alpha_ssim * ssim_loss
    
    # FaCT-GS specific: 3D Total Variation regularization
    if fact_gs_mode:
        # Voxelize a random section of the volume (e.g., 32^3)
        sampled_volume = voxelize_random_section(gaussians, volume_size=32**3)
        tv_loss = compute_3d_total_variation(sampled_volume)
        total_loss = total_loss + alpha_tv * tv_loss
        
    # 3. Backpropagation
    # Compute gradients of total_loss w.r.t. Gaussian parameters
    gradients = backward_pass(total_loss, gaussians, rendered_image)
    
    # 4. Parameter Update
    # Update Gaussian parameters using optimizer (e.g., Adam)
    gaussians = optimizer_step(gaussians, gradients)
    
    # 5. Adaptive Control (FaCT-GS specific)
    if fact_gs_mode:
        # Prune, clone, or split Gaussians based on gradients and opacity
        gaussians = adaptive_control(gaussians, gradients)
        
    return gaussians
```

**Parameters:**
*   $\alpha_{\text{SSIM}}$: Weight for SSIM loss. Not given in the sources.
*   $\alpha_{\text{TV}}$: Weight for Total Variation loss. Not given in the sources.
*   Volume sampling size: Increased from $8^3$ to $32^3$ in FaCT-GS [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Bounding Box Inefficiency:** Using square bounding boxes for Gaussian-shaped ellipses can overestimate the affected region, especially for anisotropic and transparent Gaussians, leading to computational inefficiency [S2, unofficial].
*   **Backward Pass Stalling:** Inefficient kernels can stall the backward pass, impacting training speed [S2, unofficial].

## Subtopics

### [Mini-Splatting Pipeline](9235-mini-splatting-pipeline.md)
The Mini-Splatting Pipeline is a training strategy that explicitly manages the total number of Gaussians through periodic densification, simplification, and depth-based reinitialization phases. It distinguishes itself from standard 3D Gaussian Splatting by enforcing a constrained Gaussian count to improve training efficiency and reduce memory footprint, while maintaining or improving rendering quality. See [Mini-Splatting Pipeline](9235-mini-splatting-pipeline.md).

## Operating characteristics
*   **Time Complexity:** Dominated by the rasterization and backpropagation steps, which scale with the number of Gaussians and pixels. The cubic growth in grid elements for voxelization in FaCT-GS can be computationally expensive [S2, unofficial].
*   **Memory Complexity:** Scales linearly with the number of Gaussians. FaCT-GS reduces memory footprint by pruning and controlling Gaussian count [S2, unofficial].
*   **Strengths:** Differentiable rasterization allows for end-to-end optimization of explicit 3D representations. Adaptive control enables dynamic scene representation.
*   **Limitations:** Performance bottlenecks in rasterization kernels (bounding boxes, backward pass) can impact efficiency [S2, unofficial]. Large scenes may require significant memory management [Parent].
*   **When to use:** Suitable for novel view synthesis and 3D reconstruction where high-quality rendering and explicit geometry are desired. FaCT-GS is suitable for CT reconstruction where homogeneity regularization is beneficial [S2, unofficial].

## Background
The training pipeline for 3D Gaussian Splatting was introduced as part of the 3D Gaussian Splatting method, leveraging differentiable rasterization for optimizing explicit 3D Gaussian primitives [S1, S2, unofficial]. FaCT-GS extended this pipeline for CT reconstruction, adding total variation regularization and adaptive control mechanisms [S2, unofficial].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S2] FaCT-GS: Fast and Scalable CT Reconstruction with Gaussian Splatting, https://doi.org/10.1007/978-3-032-37464-6_13, fetched 2026-10-26 (unofficial)
