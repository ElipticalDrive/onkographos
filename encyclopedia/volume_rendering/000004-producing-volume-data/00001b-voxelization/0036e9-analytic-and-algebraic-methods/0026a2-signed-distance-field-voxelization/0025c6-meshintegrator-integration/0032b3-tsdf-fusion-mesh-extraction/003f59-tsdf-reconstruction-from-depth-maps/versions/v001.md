# TSDF Reconstruction from Depth Maps

Also called: Depth Map Fusion, TSDF Reconstruction

## Overview
This method converts per-surfel depth outputs from a 2D Gaussian Splatting variant into a unified 3D mesh by computing a robust depth value per pixel and fusing the resulting depth maps via Truncated Signed Distance Function (TSDF) fusion [S1]. It differs from implicit surface extraction methods by relying on explicit depth estimation followed by volumetric integration, offering a balance between geometric fidelity and computational efficiency [S1].

## How it works
**Inputs:**
- A set of $N$ depth maps, one for each training view, where each map contains per-pixel depth values $t_i$ and transmittance values $T_i$ for all splats $i$ contributing to that pixel.
- Camera intrinsics and extrinsics for each view.

**Outputs:**
- A triangular mesh representing the reconstructed surface.

**Data Structures:**
- **TSDF Volume:** A 3D voxel grid managed by Open3D's `ScalableTSDFVolume` [S1].
  - **Voxel Size:** $0.004$ units [S1].
  - **Truncation Threshold:** $0.02$ units [S1].
  - The volume integrates signed distance values from multiple views.

**Parameters:**
- **Voxel Size:** $0.004$ [S1].
- **Truncation Threshold:** $0.02$ [S1].
- **Gradient Threshold:** $0.3$ (used in the adaptive control strategy for point cloud density, ensuring consistency with other methods) [S1].
- **Percent Dense Value:** $0.001$ (used in the adaptive control strategy) [S1].

**Steps:**

1. **Robust Depth Estimation:** For each pixel $p$ in each depth map, compute the median depth $t_{median}$. This is defined as the maximum depth $t_i$ among all splats $i$ where the transmittance $T_i$ exceeds $0.5$ [S1].
   $$ t_{median} = \max \{ t_i \mid T_i > 0.5 \} $$
   [S1]

2. **TSDF Fusion:** Project the computed $t_{median}$ depth maps into the 3D voxel space using the corresponding camera poses. Integrate these depth maps into the TSDF volume using Open3D's TSDF fusion algorithm [S1].
   - The integration uses the specified voxel size ($0.004$) and truncation threshold ($0.02$) [S1].
   - The TSDF volume accumulates weighted signed distance values from all views.

3. **Mesh Extraction:** Extract the mesh from the fused TSDF volume using Open3D's marching cubes implementation [S1]. This step is performed by the parent method [TSDF Fusion Mesh Extraction](6831-tsdf-fusion-mesh-extraction.md).

**Pitfalls:**
- **Overfitting:** The high geometric fitting capability of quadric surfels can cause overfitting in regions with sparse views or low texture, resulting in surface artifacts like dents [S1].
- **Resolution:** Evaluations are typically performed at half the original image resolution to maintain consistency with other methods and reduce computational load [S1].

## Operating characteristics
- **Time Complexity:** The method is faster than many NeRF-like methods. On the TNT dataset, full QGS achieves a mean F1-Score of 0.50 with a training time of 75 minutes [S1]. Without multi-view regularization, it requires only half the computation time of comparable methods like GOF [S1].
- **Memory Complexity:** Depends on the TSDF volume resolution and the number of views. The use of Open3D's `ScalableTSDFVolume` helps manage memory efficiently [S1].
- **Performance:**
  - **DTU Dataset:** Outperforms both implicit and explicit SOTA reconstruction methods in Chamfer Distance [S1].
  - **TNT Dataset:** Achieves state-of-the-art F1-Score (0.50 mean) [S1].
  - **Mip-NeRF 360 Indoor:** PSNR 30.45, SSIM 0.919, LPIPS 0.184 [S1].
  - **Mip-NeRF 360 Outdoor:** PSNR 24.32, SSIM 0.706, LPIPS 0.242 [S1].
- **Strengths:** Superior geometric fitting capability of quadrics enables finer detail preservation [S1]. Comparable accuracy to GOF with significantly lower computation time [S1].
- **Limitations:** Susceptible to artifacts in sparse/low-texture regions due to overfitting [S1].

## Background
This method is part of the Quadratic Gaussian Splatting (QGS) pipeline, which uses second-order geometric primitives for high-quality surface reconstruction [S1]. It builds on the TSDF fusion approach used in 2DGS [S1]. The adaptive control strategy for point cloud density is adopted from 3DGS [S1]. Experiments were conducted on a single A6000 GPU [S1].

## Sources
[S1] Quadratic Gaussian Splatting: High Quality Surface Reconstruction with Second-Order Geometric Primitives, https://doi.org/10.1109/ICCV51701.2025.02624, fetched 2026-10-30
