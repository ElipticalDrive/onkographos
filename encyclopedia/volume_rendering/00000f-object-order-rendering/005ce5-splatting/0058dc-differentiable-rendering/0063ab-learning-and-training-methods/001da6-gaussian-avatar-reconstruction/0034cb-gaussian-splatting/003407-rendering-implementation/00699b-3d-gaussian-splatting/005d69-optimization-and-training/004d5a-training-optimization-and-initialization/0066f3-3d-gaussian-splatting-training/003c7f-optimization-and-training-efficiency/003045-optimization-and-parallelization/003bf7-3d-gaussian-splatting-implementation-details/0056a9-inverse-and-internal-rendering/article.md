# Inverse and Internal Rendering

Also called: GS-IR, Inner 3D Gaussian Splatting, Internal View Synthesis, Volumetric Plane-Based Gaussian Selection

## Overview
Inverse and Internal Rendering extends 3D Gaussian Splatting (3DGS) to solve problems where standard surface-based supervision is insufficient: estimating physical properties from unknown lighting (Inverse Rendering) and reconstructing occluded internal structures from sparse, pose-free data (Internal Rendering) [S1]. Unlike standard 3DGS which optimizes for camera-visible surfaces, this family enables photorealistic relighting via baked occlusion caches [S1] and volumetric inference via conditional 2D splatting or plane-based Gaussian selection [S1].

## Shared principles
The methods share the core 3D Gaussian representation defined in [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md), where each primitive $i$ is defined by mean position $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, opacity $\alpha_i$, and color $c_i$ [S1]. The rendering equation for a pixel remains the standard alpha-compositing formula:

$$ I(u, v, t) = \sum_{i=1}^{N} p_i(u, v, t)\alpha_i c_i \prod_{j=1}^{i-1} (1 - p_j(u, v, t)\alpha_j) $$

where $p_i$ is the screen-space density contribution [S1]. However, the optimization targets and geometric constraints diverge significantly from standard surface reconstruction.

## Subtopics

### GS-IR (Gaussian Splatting for Internal Rendering)
GS-IR is a plane-based internal rendering framework that replaces exterior-view supervision with cross-sectional supervision sampled from a volume to reconstruct internal volumetric structure [S1]. It is applied to 3DGS, 2DGS, and Mip-Splatting without modifying their rasterizers [S1].

**Definition and Steps:**
1.  Virtual cameras define slicing planes within the volume [S1].
2.  A binary mask selects the Gaussian subset whose centers lie within the plane thickness [S1].
3.  The selected Gaussian subset is rendered using the original Gaussian rasterizer [S1].
4.  Optimization is performed against ground-truth cross-sections generated through trilinear resampling [S1].
5.  Distance-aware soft weighting is applied to continuously attenuate Gaussian opacity as the distance from the Gaussian to the plane increases [S1].

**Parameters:**
*   **Plane thickness:** The spatial extent within which Gaussian centers are selected by the binary mask [S1].

**Performance:**
*   Improves PSNR by up to 21.52 dB and SSIM by up to 0.54 on KiTS23 CT and IXI MRI datasets [S1].

**Comparison:**
*   Unlike conventional 3DGS which constrains camera-visible surfaces, GS-IR focuses optimization gradients on interior Gaussians to supervise occluded internal regions [S1].

### 3DGS Update Procedure with Albedo-Shading Decomposition
This procedure efficiently captures global illumination by updating 3D Gaussians using path-traced renderings, combined with albedo-shading decomposition [S2, unofficial].

**Steps:**
1.  Utilize path-traced renderings to efficiently update 3DGS and capture global illumination effects [S2, unofficial].
2.  Apply albedo-shading decomposition strategy to separate intrinsic scene properties for efficient global illumination capture [S2, unofficial].

## Background
Standard 3D Gaussian Splatting (3DGS) represents scenes with anisotropic Gaussian primitives optimized from multi-view images and rendered through differentiable rasterization and alpha compositing [S1]. Conventional 3DGS training primarily constrains camera-visible surfaces, whereas occluded interior regions remain insufficiently supervised [S1]. In medical imaging, internal view synthesis requires cross-sectional images from volumetric CT and MRI datasets at arbitrary plane orientations and spatial positions [S1]. GS-IR investigates whether Gaussian primitives can reconstruct internal volumetric structure under cross-sectional supervision [S1].

For inverse rendering, NeRF-based methods use implicit neural representations and ray tracing (backward mapping), which suffer from low expressive power and high computational complexity [S1]. GS-IR addresses the specific challenges of applying 3DGS to inverse rendering: the lack of native plausible normal generation and the inability of forward mapping to trace occlusion for indirect lighting [S1]. It estimates scene geometry, surface material, and environment illumination from multi-view images captured under unknown lighting conditions [S1].

Inner 3D Gaussian Splatting (InnerGS) solves the problem of reconstructing internal volumes from slice-based inputs (e.g., medical imaging or material scans) where camera poses and projection matrices are unavailable [S1]. Unlike standard 3DGS, which relies on camera projection and tends to concentrate Gaussians on object surfaces leaving interiors empty, InnerGS uses **Conditional Splatting** to adaptively determine Gaussian influence per depth slice [S1]. It explicitly factorizes the 3D density into conditional 2D splats and a marginal 1D depth Gaussian, enabling tile-based rasterization without external camera parameters [S1].

## Sources
[S1] Volumetric Plane-Based Gaussian Selection for Internal View Synthesis With 3-D Gaussian Splatting, https://doi.org/10.1109/access.2026.3737246, fetched 2026-10-27
[S2] Mesh2GS: White-Box 3DGS Construction via Plenoptic Sampling, https://doi.org/10.48550/arXiv.2606.21898, fetched 2026-10-30 (unofficial)
