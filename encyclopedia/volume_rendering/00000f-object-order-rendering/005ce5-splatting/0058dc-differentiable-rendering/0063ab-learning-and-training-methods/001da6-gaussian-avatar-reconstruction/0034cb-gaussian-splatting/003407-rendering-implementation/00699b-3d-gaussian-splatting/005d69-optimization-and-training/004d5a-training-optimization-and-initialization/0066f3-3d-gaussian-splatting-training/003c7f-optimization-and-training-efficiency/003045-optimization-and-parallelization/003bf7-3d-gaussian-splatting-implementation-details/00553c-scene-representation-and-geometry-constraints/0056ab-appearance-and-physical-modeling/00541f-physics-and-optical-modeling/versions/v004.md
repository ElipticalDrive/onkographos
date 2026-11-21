# Physics and Optical Modeling


## Overview
Physics and Optical Modeling extends the standard 3D Gaussian Splatting representation to handle non-vacuum media and physically consistent surface extraction by modifying the rendering equation or the underlying density field [G]. It solves the problem of light transport in scattering media and the geometric inconsistency of discrete alpha-blending for surface reconstruction [G]. Unlike [Color and Appearance Modeling](14740-color-and-appearance-modeling.md), which focuses on view-dependent shading in clear space, this family explicitly models volumetric attenuation and backscatter or reformulates the Gaussian primitive as a continuous attenuation field [G].

## Shared principles
This family inherits the standard 3DGS representation and rendering pipeline from [Appearance and Physical Modeling](15766-appearance-and-physical-modeling.md).

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
**What it is:** A method that reformulates rasterization-based 3DGS into a volumetric ray-marching framework by mapping discrete Gaussians to a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$ [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].
**Distinction:** Unlike standard alpha-blending which assumes negligible 3D overlap, this model explicitly defines a view-dependent density that preserves ellipsoidal geometry, enabling unbiased surface extraction [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].
**When to choose:** When multi-view consistent depth maps and surface extraction are required, avoiding the surface erosion caused by standard alpha-blending approximations [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].
**Link:** [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
**What it is:** An adaptation of 3DGS for underwater environments that learns direct attenuation ($B_d$), infinite backscatter color ($B_\infty$), and backscatter coefficient ($B_b$) [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].
**Distinction:** It explicitly models the physical interaction of light with water (color cast and visibility loss), which vanilla 3DGS fails to handle, while remaining significantly faster than NeRF-based underwater alternatives [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].
**When to choose:** For 3D reconstruction in aquatic scenes where color distortion and attenuation are dominant [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].
**Link:** [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)

### Backscatter Estimation and Optimization
**Step:**
1. Divide the depth map (from the current scene estimate) into 10 evenly-spaced clusters covering the range from $\min(z)$ to $\max(z)$ [S1].
2. Within each cluster, identify pixels in the ground-truth image whose RGB triplets are below the bottom 1st percentile [S1].
3. These triplets serve as an overestimate of the backscatter signal $B_s$ [S1].
4. Use nonlinear least squares fitting on pairs $(B_s, z)$ to estimate parameters $B_\infty$ and $B_b$ [S1].
5. Re-estimate backscatter parameters $B_\infty$ and $B_b$ every 500 optimization steps to guide convergence toward the medium coefficients [S1].

**Formula:**
The base loss function is:
$$ L = \sum_{k=1}^K (1 - \lambda_1) \| I_k^{gt} - I_k^r(\Theta) \|_{\ell_1} + \lambda_1 L_{D-SSIM}(I_k^{gt}, I_k^r(\Theta)) $$
where $K$ is the number of images and $\lambda_1 \in [0,1]$ is user-defined [S1].

**Parameter:**
Backscatter parameter bounds are $B_\infty \in [0, 1]$ and $B_b \in [0, 5]$ [S1].

**Pitfall:**
The camera sensor must have a linear response to light. In-camera processed .jpg images or gamma-corrected images cannot be used unless non-linearities can be reversed [S1].

## Background
The continuous attenuation model was introduced to address the geometric inconsistencies of discrete Gaussian splatting for surface extraction, leveraging theoretical results from Orienting Attenuation (OaV) [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md]. Underwater 3D Gaussian Splatting (Gaussian Splashing) was developed to extend 3DGS to aquatic environments by incorporating ocean optics models, specifically addressing the limitations of vanilla 3DGS in handling distance-dependent attenuation and color casts [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].

## Sources
[S1] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
