# GaussianShader Material Model

Also called: GaussianShader

## Overview
GaussianShader enhances 3D Gaussian Splatting (3DGS) by incorporating trainable material parameters—diffuse color, specular intensity, roughness, and surface normal—along with a global environment map to achieve physically-based rendering (PBR) [S1]. It distinguishes itself from standard 3DGS by disentangling geometry from material properties, allowing for accurate specular reflections and relighting capabilities [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images of a 3D scene [S1].
*   **Outputs:** A rendered image and optimized 3D Gaussian parameters [S1].

**Data Structures:**
Each Gaussian kernel $p$ stores the following trainable parameters:
*   Position: $\mathbf{x}_p$
*   Opacity: $\sigma_p$
*   Covariance: $\mathbf{A}_p$
*   Diffuse color: $\mathbf{d}_p$
*   Specular intensity: $\mathbf{s}_p$
*   Roughness: $\rho_p$
*   Surface normal: $\mathbf{n}_p$
*   Global Environment Map: $\mathcal{E}$ (pre-filtered into mip maps) [S1].

**Rendering Formula:**
The color $\mathbf{c}_p$ for a kernel is computed as [S1]:
$$ \mathbf{c}_p(\mathbf{r}_i) = \mathbf{d}_p + \mathbf{s}_p \odot \mathcal{L}_s(\mathbf{r}_i, \mathbf{n}_p, \rho_p) $$
Where:
*   $\mathbf{r}_i$ is the view direction from the camera to pixel $i$ [S1].
*   $\odot$ denotes element-wise multiplication [S1].
*   $\mathcal{L}_s(\mathbf{r}_i, \mathbf{n}_p, \rho_p)$ is the specular light contribution for the kernel along $\mathbf{r}_i$, dependent on the normal and roughness [S1]. This term is pre-filtered into multiple mip maps [S1].

**Algorithm Steps:**
1.  **Preprocessing:** Separate foreground objects and background from input images. Extract the surface of masked objects [S1].
2.  **Initialization:** Initialize 3D Gaussian kernels with position, opacity, covariance, and material parameters ($\mathbf{d}_p, \mathbf{s}_p, \rho_p, \mathbf{n}_p$) [S1].
3.  **Shading:** For each pixel, determine the view direction $\mathbf{r}_i$. Compute the kernel color $\mathbf{c}_p$ using the PBR formula above, utilizing the pre-filtered environment map for $\mathcal{L}_s$ [S1].
4.  **Splatting:** Project the 3D Gaussians to 2D screen space. Calculate the 2D Gaussian weight $G_k(i)$ for the $k$-th kernel at pixel $i$ based on the kernel center's depth and covariance [S1].
5.  **Compositing:** Accumulate colors front-to-back using the standard 3DGS alpha blending process [S1].
6.  **Optimization:** Compare the rendered image with the ground truth training view. Compute a loss function and back-propagate to update $\mathbf{x}_p, \sigma_p, \mathbf{A}_p, \mathbf{d}_p, \mathbf{s}_p, \rho_p, \mathbf{n}_p$ [S1].
7.  **Constraint (Optional):** Apply normal orientation constraints to handle challenging reflective scenes, similar to Ref-NeRF [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Over-stretching:** Gaussian kernels may over-stretch near solid object surfaces, causing rendering artifacts. An anisotropy loss can be applied to mitigate this [S1].
*   **Lighting Overfitting:** Geometry attributes (like opacity) may overfit to input lighting conditions, leading to blending artifacts when relit. This method avoids some such issues by explicitly modeling material parameters, though deferred shading techniques (as in DeferredGS) are noted as a separate solution to this specific problem [S2, unofficial].

## Subtopics

### RelightableGaussian
A method that separately models texture and lighting [S2, unofficial]. It defines additional material parameters on each Gaussian to represent texture and approximates lighting using a learnable environment map [S2, unofficial].

### GIR (Inverse Rendering with Gaussians)
Shares the same disentanglement paradigm as GaussianShader by binding material parameters to 3D Gaussians [S2, unofficial]. It adds normal orientation constraints to Gaussians to deal with more challenging reflective scenes, similar to Ref-NeRF [S2, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Strengths:** Provides visually accurate representations of specular reflections and simulated interactions between solids and fluids [S1]. Allows for independent editing of texture and lighting [S2, unofficial].
*   **Limitations:** Requires optimization of additional material parameters per kernel, increasing training complexity compared to standard 3DGS [S1].

## Background
GaussianShader was introduced to enhance 3DGS with physically-based rendering capabilities [S1]. It builds on the 3DGS pipeline and Position-Based Dynamics (PBD) [S1]. The method is cited in the context of unified particles for motion synthesis and rendering [S1].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
