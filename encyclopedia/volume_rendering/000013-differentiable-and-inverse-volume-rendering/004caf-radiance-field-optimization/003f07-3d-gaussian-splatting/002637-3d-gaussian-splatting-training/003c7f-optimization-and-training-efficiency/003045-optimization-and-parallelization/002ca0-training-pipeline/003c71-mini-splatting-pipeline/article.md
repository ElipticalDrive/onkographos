# Mini-Splatting Pipeline

Also called: Mini-Splatting, Mini-Splatting-D, Mini-Splatting-C

## Overview
The Mini-Splatting Pipeline is a training strategy for 3D Gaussian Splatting that explicitly manages the total number of Gaussians through periodic densification, simplification, and depth-based reinitialization phases [S1]. It distinguishes itself from standard 3D Gaussian Splatting by enforcing a constrained Gaussian count via a manually controlled sampling ratio [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse Structure-from-Motion (SfM) points, camera poses, and multi-view images [S1].
*   **Output:** A set of optimized 3D Gaussians and, for the compression variant, a compressed model representation [S1].

**Data Structures:**
The pipeline maintains a dynamic list of Gaussians [S1]. Each Gaussian $g$ is defined by standard 3D Gaussian Splatting attributes: center $\mathbf{p} \in \mathbb{R}^3$, scale $\mathbf{s} \in \mathbb{R}^3$ (or rotation quaternion $\mathbf{q}$ and scale vector), opacity $\alpha \in [0, 1]$, and spherical harmonics (SH) coefficients $\mathbf{c}$ [G].

**Parameters:**
*   `DensificationIteration`: 15,000 [S1].
*   `SimplificationIteration1`: 15,000 [S1].
*   `SimplificationIteration2`: 20,000 [S1].
*   `Depth Reinitialization Interval`: Every 5,000 iterations [S1].
*   `Sampling Ratio`: Manually controlled parameter affecting the final Gaussian count; high ratios may cause background distortion [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the set of Gaussians from SfM points [S1, Line 1]. Set iteration counter $i = 0$ [S1, Line 2].
2.  **Loop:** While not converged [S1, Line 3]:
    a.  **Render and Loss:** Render the current Gaussian set and compute loss against ground truth images [G].
    b.  **Densification:** If $i$ is a multiple of `DensificationIteration` (15K), perform densification. The design of importance metrics for this step is case-dependent and hand-crafted, presented as an experimental trick in the appendix [S1].
    c.  **Depth Reinitialization:** If $i$ is a multiple of 5,000, perform depth-based reinitialization [S1].
    d.  **Simplification:** If $i = \text{SimplificationIteration1}$ (15K), perform simplification [S1, Lines 19-21]. If $i = \text{SimplificationIteration2}$ (20K), perform simplification again [S1, Lines 19-21].
    e.  **Increment:** $i \leftarrow i + 1$ [S1, Line 24].

**Variants:**
*   **Mini-Splatting-D:** Disables the simplification process entirely [S1].
*   **Mini-Splatting-C:** Applies compression to the trained model. Gaussian centers are stored as `float32`. Other attributes (scales, SH coefficients) are processed using Region-adaptive Hierarchical Transform (RHT) [6] with a depth level of 16 and a quantization step of 0.02. The transformed coefficients are then losslessly compressed using `numpy.savez_compressed()` [S1].

**Pitfalls:**
*   **Sky Areas:** The depth-based reinitialization strategy fails in areas without reliable depth values, such as the sky [S1].
*   **Background Distortion:** A high sampling ratio leads to distortion in the background [S1].
*   **Manual Tuning:** The number of Gaussians is manually controlled by the sampling ratio, and finding the minimal number for high-quality rendering remains a challenge [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mini_splatting_pipeline(sfm_points, images, camera_poses):
    # Helper definitions:
    # initialize_from_sfm: Standard 3DGS initialization from SfM points [G]
    # check_convergence: Standard convergence check based on loss/iterations [G]
    # render_gaussians: Differentiable rasterization of Gaussians [G]
    # compute_loss: Standard loss computation (e.g., L1 + SSIM) [G]
    # densify_gaussians: Split/clone Gaussians based on importance metrics [G]
    # depth_reinitialize_gaussians: Reinitialize Gaussians using depth maps [G]
    # simplify_gaussians: Remove/merge low-importance Gaussians [G]
    
    gaussians = initialize_from_sfm(sfm_points)
    i = 0
    densification_interval = 15000
    simplification_iter1 = 15000
    simplification_iter2 = 20000
    reinit_interval = 5000
    
    while not check_convergence(gaussians, images):
        rendered_images = render_gaussians(gaussians, camera_poses)
        loss = compute_loss(rendered_images, images)
        loss.backward()
        
        if i % densification_interval == 0:
            # Importance metrics are case-dependent and hand-crafted; see appendix [S1]
            gaussians = densify_gaussians(gaussians)
            
        if i % reinit_interval == 0:
            # Fails in sky regions without reliable depth values [S1]
            gaussians = depth_reinitialize_gaussians(gaussians)
            
        if i == simplification_iter1:
            gaussians = simplify_gaussians(gaussians)
        if i == simplification_iter2:
            gaussians = simplify_gaussians(gaussians)
            
        i += 1
        
    return gaussians
```

## Operating characteristics
*   **Performance:** On the Mip-NeRF 360 dataset, Mini-Splatting achieves the following metrics:
    *   **SSIM:** bicycle 0.772, flowers 0.626, garden 0.847, stump 0.806, treehill 0.654, room 0.921, counter 0.904, kitchen 0.926, bonsai 0.940, truck 0.874, train 0.796, johnson 0.912, playroom 0.912 [S1].
    *   **PSNR:** bicycle 25.15, flowers 21.58, garden 26.84, stump 27.26, treehill 22.72, room 31.39, counter 28.41, kitchen 31.17, bonsai 31.57, truck 25.09, train 21.28, johnson 29.47, playroom 30.49 [S1].
    *   **LPIPS:** bicycle 0.225, flowers 0.327, garden 0.150, stump 0.199, treehill 0.312, room 0.211, counter 0.199, kitchen 0.130, bonsai 0.199, truck 0.159, train 0.245, johnson 0.257, playroom 0.250 [S1].
*   **Comparison:** Mini-Splatting demonstrates consistent improvement compared to standard 3D Gaussian Splatting (3DGS) in most scans and surpasses Zip-NeRF in SSIM and LPIPS metrics on certain datasets [S1].
*   **Strengths:** Reduced Gaussian count leads to lower memory usage. The compression variant (Mini-Splatting-C) further reduces storage size [S1].
*   **Limitations:** Manual tuning of the sampling ratio is required. Depth-based reinitialization fails in sky regions. High sampling ratios can distort backgrounds [S1].

## Background
Mini-Splatting was introduced in the paper "Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians" [S1]. It builds on the foundational 3D Gaussian Splatting framework [Parent](5325-training-pipeline.md) [S1].

## Sources
[S1] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
