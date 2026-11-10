# Mini-Splatting Pipeline

Also called: Mini-Splatting, Mini-Splatting-D, Mini-Splatting-C

## Overview
The Mini-Splatting Pipeline is a training strategy for 3D Gaussian Splatting that explicitly manages the total number of Gaussians through periodic densification, simplification, and depth-based reinitialization phases [S1]. It distinguishes itself from standard 3D Gaussian Splatting by enforcing a constrained Gaussian count to improve training efficiency and reduce memory footprint, while maintaining or improving rendering quality [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse Structure-from-Motion (SfM) points, camera poses, and multi-view images.
*   **Output:** A set of 3D Gaussians with optimized attributes (position, covariance, opacity, spherical harmonics coefficients) and a compressed model representation (for Mini-Splatting-C).

**Data Structures:**
The pipeline maintains a dynamic list of Gaussians. Each Gaussian $g$ is defined by:
*   Center $\mathbf{p} \in \mathbb{R}^3$
*   Scale $\mathbf{s} \in \mathbb{R}^3$ (or rotation quaternion $\mathbf{q}$ and scale vector)
*   Opacity $\alpha \in [0, 1]$
*   Spherical Harmonics (SH) coefficients $\mathbf{c}$

**Parameters:**
*   `DensificationIteration`: 15,000 [S1].
*   `SimplificationIteration1`: 15,000 [S1].
*   `SimplificationIteration2`: 20,000 [S1].
*   `Depth Reinitialization Interval`: Every 5,000 iterations [S1].
*   `Sampling Ratio`: Manually controlled parameter affecting the final Gaussian count; high ratios may cause background distortion [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the set of Gaussians from SfM points [S1, Line 1]. Set iteration counter $i = 0$ [S1, Line 2].
2.  **Loop:** While not converged:
    a.  **Render and Loss:** Render the current Gaussian set and compute loss against ground truth images.
    b.  **Densification:** If $i$ is a multiple of `DensificationIteration` (15K), perform densification. This involves splitting and cloning Gaussians based on importance metrics (e.g., gradient magnitude or intersection counts) to increase density in under-represented areas [S1, Lines 15-17]. The specific importance metrics used are described as experimental tricks in the source appendix, but generally involve intersection-preserving sampling [S1].
    c.  **Depth Reinitialization:** If $i$ is a multiple of 5,000, perform depth-based reinitialization. This strategy reinitializes Gaussian models based on depth information to correct drift or improve geometric consistency [S1].
    d.  **Simplification:**
        i.  If $i = \text{SimplificationIteration1}$ (15K), perform simplification. This reduces the number of Gaussians by removing low-importance ones or merging them, using intersection-preserving sampling [S1, Lines 19-21].
        ii. If $i = \text{SimplificationIteration2}$ (20K), perform simplification again [S1, Lines 19-21].
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
    # Initialization
    gaussians = initialize_from_sfm(sfm_points)
    i = 0
    densification_interval = 15000
    simplification_iter1 = 15000
    simplification_iter2 = 20000
    reinit_interval = 5000
    
    while not converged(gaussians, images):
        # Render and compute loss
        rendered_images = render(gaussians, camera_poses)
        loss = compute_loss(rendered_images, images)
        loss.backward()
        
        # Densification
        if i % densification_interval == 0:
            gaussians = densify(gaussians, importance_metric)
            
        # Depth Reinitialization
        if i % reinit_interval == 0:
            gaussians = depth_reinitialize(gaussians, depth_maps)
            
        # Simplification
        if i == simplification_iter1:
            gaussians = simplify(gaussians, intersection_preserving_sampling)
        if i == simplification_iter2:
            gaussians = simplify(gaussians, intersection_preserving_sampling)
            
        i += 1
        
    return gaussians
```

## Operating characteristics
*   **Performance:** On the Mip-NeRF 360 dataset, Mini-Splatting achieves the following metrics:
    *   **SSIM:** bicycle 0.772, flowers 0.626, garden 0.847, stump 0.806, treehill 0.654, room 0.921, counter 0.904, kitchen 0.926, bonsai 0.940, truck 0.874, train 0.796, johnson 0.912, playroom 0.912 [S1].
    *   **PSNR:** bicycle 25.15, flowers 21.58, garden 26.84, stump 27.26, treehill 22.72, room 31.39, counter 28.41, kitchen 31.17, bonsai 31.57, truck 25.09, train 21.28, johnson 29.47, playroom 30.49 [S1].
    *   **LPIPS:** bicycle 0.225, flowers 0.327, garden 0.150, stump 0.199, treehill 0.312, room 0.211, counter 0.199, kitchen 0.130, bonsai 0.199, truck 0.159, train 0.245, johnson 0.257, playroom 0.250 [S1].
*   **Comparison:** Mini-Splatting demonstrates consistent improvement compared to standard 3D Gaussian Splatting (3DGS) in most scans and surpasses Zip-NeRF in SSIM and LPIPS metrics on certain datasets [S1].
*   **Strengths:** Reduced Gaussian count leads to lower memory usage and potentially faster training/inference. The compression variant (Mini-Splatting-C) further reduces storage size.
*   **Limitations:** Manual tuning of the sampling ratio is required. Depth-based reinitialization fails in sky regions. High sampling ratios can distort backgrounds [S1].

## Background
Mini-Splatting was introduced to address the high memory and computational costs associated with the large number of Gaussians in standard 3D Gaussian Splatting [S1]. The method was proposed by the authors of the "Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians" paper [S1]. It builds on the foundational 3D Gaussian Splatting framework [Parent](5325-training-pipeline.md).

## Sources
[S1] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
