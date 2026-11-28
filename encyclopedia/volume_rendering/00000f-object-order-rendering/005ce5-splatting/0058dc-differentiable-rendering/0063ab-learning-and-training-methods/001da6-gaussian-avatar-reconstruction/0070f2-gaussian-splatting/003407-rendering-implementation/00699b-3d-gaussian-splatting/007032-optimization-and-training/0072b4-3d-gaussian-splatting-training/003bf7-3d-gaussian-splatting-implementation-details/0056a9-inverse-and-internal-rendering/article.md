# Inverse and Internal Rendering

Also called: GS-IR, Inner 3D Gaussian Splatting, Gradient Calculation for Global Illumination

## Overview
Inverse and Internal Rendering extends 3D Gaussian Splatting (3DGS) to handle scenarios where standard surface-based supervision is insufficient or unavailable, specifically targeting internal volumetric structures, unknown lighting conditions, and global illumination effects. It solves the problem that conventional 3DGS optimizes only camera-visible surfaces, leaving occluded interiors empty, and relies on known camera poses and direct lighting. This family sets itself apart from standard 3DGS by introducing plane-based cross-sectional supervision for internal reconstruction [S1], pose-free conditional splatting for sliced data [15745-inner-3d-gaussian-splatting.md], and analytical gradient propagation through global illumination solvers for material and lighting optimization [23297-gradient-calculation-for-global-illumination.md].

## Shared principles
The methods in this family share the core 3D Gaussian representation defined in [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md). Each primitive $i$ is defined by:
*   Mean position $\mu_i \in \mathbb{R}^3$
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, decomposed into scaling $S$ and rotation $R$ such that $\Sigma = RSS^\top R^\top$
*   Opacity $\alpha_i$
*   Color $c_i$, often modeled via Spherical Harmonics (SH) coefficients

The rendering equation for a pixel $(u, v)$ at depth $t$ (or slice) is:
$$ I(u, v, t) = \sum_{i=1}^{N} p_i(u, v, t)\alpha_i c_i \prod_{j=1}^{i-1} (1 - p_j(u, v, t)\alpha_j) $$
where $p_i(u, v, t)$ is the screen-space density contribution of the $i$-th Gaussian, computed as:
$$ p(x) = e^{-\frac{1}{2} (x-\mu)^\top \Sigma^{-1} (x-\mu)} $$
Optimization is performed by minimizing a loss function $\mathcal{L}$ against target data (images, cross-sections, or radiance gradients), requiring differentiable rendering pipelines that support backpropagation through the splatting and compositing operations.

## Subtopics

### [GS-IR](14852-gs-ir.md)
GS-IR is an inverse rendering approach based on 3D Gaussian Splatting that leverages forward mapping volume rendering to achieve photorealistic novel view synthesis and relighting results, estimating scene geometry, surface material, and environment illumination from multi-view images captured under unknown lighting conditions. It sets itself apart from NeRF-based inverse rendering methods by using 3D Gaussians and forward mapping splatting for real-time rendering efficiency, avoiding the low expressive power and high computational complexity of implicit neural representations. Unlike sibling methods that may rely on ray tracing for occlusion, GS-IR handles occlusion via a baking-based cache, making it available for both objects and unbounded scenes.

### [Inner 3D Gaussian Splatting](15745-inner-3d-gaussian-splatting.md)
Inner 3D Gaussian Splatting (InnerGS) is a framework for direct volumetric inference that models continuous volumetric density using 3D Gaussians to reconstruct internal structures from sparse, pose-free sliced data. It solves the problem that standard 3D Gaussian Splatting (3DGS) relies on camera parameters (view/projection matrices) which are absent in sliced data, and that learned Gaussians concentrate on the object's surface, leaving the interior volume empty. It sets itself apart from siblings like [GS-IR](14852-gs-ir.md) by eliminating the need for camera poses and using a slice-based rendering pipeline centered on Conditional Splatting, which dynamically adapts the sampling of Gaussians for each 2D slice.

### [Gradient Calculation for Global Illumination](23297-gradient-calculation-for-global-illumination.md)
This method computes the analytical gradients of a loss function $\mathcal{L}$ with respect to the underlying geometry, materials, and lighting conditions of a scene represented by 3D Gaussian surfels. It solves the problem of efficiently backpropagating through the global illumination (light transport) solver used in differentiable rendering. Unlike automatic differentiation techniques such as path replay backpropagation, this method exploits the duality between the forward pass (light propagation) and the backward pass (gradient propagation), allowing for non-recursive, accurate, and efficient analytical definitions of gradients for emissions, materials, and geometry.

#### GS-IR (Gaussian Splatting for Internal Rendering)
**Definition**
GS-IR is a plane-based internal rendering framework that replaces exterior-view supervision with cross-sectional supervision sampled from a volume to reconstruct internal volumetric structure [S1].

**Steps**
1.  Virtual cameras define slicing planes within the volume [S1].
2.  A binary mask selects the Gaussian subset whose centers lie within the plane thickness [S1].
3.  The selected Gaussian subset is rendered using the original Gaussian rasterizer [S1].
4.  Optimization is performed against ground-truth cross-sections generated through trilinear resampling [S1].
5.  Distance-aware soft weighting is applied to continuously attenuate Gaussian opacity as the distance from the Gaussian to the plane increases [S1].

**Parameters**
*   Plane thickness: The spatial extent within which Gaussian centers are selected by the binary mask [S1].

**Performance**
GS-IR improves PSNR by up to 21.52 dB and SSIM by up to 0.54 on KiTS23 CT and IXI MRI datasets [S1].

**Comparison**
Unlike conventional 3DGS which constrains camera-visible surfaces, GS-IR focuses optimization gradients on interior Gaussians to supervise occluded internal regions [S1].

**Usage**
GS-IR is applied to 3DGS, 2DGS, and Mip-Splatting without modifying their rasterizers [S1].

#### 3DGS Update Procedure with Albedo-Shading Decomposition
**Definition**
A procedure to efficiently capture global illumination by updating 3D Gaussians using path-traced renderings, combined with albedo-shading decomposition [S2, unofficial].

**Steps**
1.  Utilize path-traced renderings to efficiently update 3DGS and capture global illumination effects [S2, unofficial].
2.  Apply albedo-shading decomposition strategy to separate intrinsic scene properties for efficient global illumination capture [S2, unofficial].

## Background
The concept of inverse rendering with 3D Gaussians was introduced by Choishroy et al. in "Volumetric Plane-Based Gaussian Selection for Internal View Synthesis With 3-D Gaussian Splatting" (2026) [S1], addressing the lack of internal supervision in standard 3DGS. The method for analytical gradient calculation for global illumination in Gaussian surfels was detailed in "Gradient Calculation for Global Illumination" [S1]. The update procedure with albedo-shading decomposition was proposed in "Mesh2GS: White-Box 3DGS Construction via Plenoptic Sampling" (2026) [S2, unofficial]. These methods are used in medical imaging (CT/MRI reconstruction), real-time shared rendering, and high-fidelity scene relighting.

## Sources
[S1] Volumetric Plane-Based Gaussian Selection for Internal View Synthesis With 3-D Gaussian Splatting, https://doi.org/10.1109/access.2026.3737246, fetched 2026-10-27
[S2] Mesh2GS: White-Box 3DGS Construction via Plenoptic Sampling, https://doi.org/10.48550/arXiv.2606.21898, fetched 2026-10-30 (unofficial)
