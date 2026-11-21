# 3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)


## Overview
3iGS enhances 3D Gaussian Splatting by modeling outgoing radiance as a function of a continuous local illumination field and individual Gaussian BRDF attributes, rather than optimizing outgoing radiance in isolation [S1]. It solves the problem of poor specular view-dependent effects in standard 3DGS by introducing a compact, factorized tensor illumination grid and a neural shading network, significantly improving rendering quality for reflective surfaces while maintaining real-time rasterization speeds [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D Gaussian parameters (mean $\mu$, opacity $\alpha$, covariance $\Sigma$), initial SH coefficients, camera parameters (view matrix $W$, affine projection $J$), and training images.
*   **Outputs:** Rendered image pixels $\hat{\mathbf{C}}$.

**Data Structures:**
1.  **3D Gaussians:** Each Gaussian $i$ stores:
    *   Mean $\mu_i \in \mathbb{R}^3$.
    *   Opacity $\alpha_i \in \mathbb{R}$.
    *   Scaling matrix $S_i$ and Rotation matrix $R_i$ (used to derive covariance $\Sigma_i = R_i S_i S_i^T R_i^T$) [S1].
    *   Diffuse color $\mathbf{c}_{d,i} \in \mathbb{R}^3$.
    *   BRDF features $\rho_i$: Repurposed 16x3 SH coefficients from 3DGS, plus 4 additional parameters for base color and roughness [S1].
2.  **Illumination Grid $\mathcal{G}_l$:** A learnable factorized tensor field mapping spatial coordinates to incoming illumination $L_i$.
    *   Resolution: Coarse grid of $150^3$ voxels [S1].
    *   Representation: Vector-Matrix spatial factorization with rank $R_L$ and feature channels $R_L$.
    *   Components: Feature tensors $\mathbf{A}_{L,r}^X, \mathbf{A}_{L,r}^Y, \mathbf{A}_{L,r}^Z$ and feature vectors $\mathbf{b}_{3r-2}, \mathbf{b}_{3r-1}, \mathbf{b}_{3r}$ for $r=1 \dots R_L$ [S1].
3.  **Neural Renderer $\mathcal{F}$:** A small neural network mapping $\{\rho_i, L_i, \omega_o\} \mapsto \mathbf{c}_{s,i}$ [S1].

**Numbered Steps:**

1.  **Initialize Gaussians:** Fit 3D Gaussians to the scene point cloud (e.g., from SfM) [S1]. Initialize the illumination grid $\mathcal{G}_l$ with a boundary encapsulating the scene bounding box [S1].
2.  **Compute Covariance in Camera Space:** For each Gaussian, compute the covariance matrix in camera coordinates:
    $$ \mathbf{\Sigma}' = \mathbf{JW\Sigma W^TJ^T} $$
    where $\mathbf{\Sigma} = \mathbf{RSS^TR^T}$, $W$ is the viewing transformation matrix, and $J$ is the affine approximation of the projective transformation [S1].
3.  **Query Illumination Field:** For each Gaussian $i$ at mean $\mathbf{x}_i$, compute incoming illumination $L_i$ by interpolating from the factorized tensor grid $\mathcal{G}_l$:
    $$ \mathcal{G}_l : \{\mathbf{x}_i\} \mapsto L_i $$
    The grid value is calculated via vector-matrix spatial factorization [S1]:
    $$ \mathcal{G}_l = \sum_{r=1}^{R_L} \mathbf{A}_{L,r}^X \circ \mathbf{b}_{3r-2} + \mathbf{A}_{L,r}^Y \circ \mathbf{b}_{3r-1} + \mathbf{A}_{L,r}^Z \circ \mathbf{b}_{3r} $$
    where $\circ$ denotes the appropriate tensor contraction/interaction defined by the factorization scheme (specifically, combining the spatial feature tensors with the appearance feature vectors) [S1].
4.  **Compute Specular Color:** Encode the viewing direction $\omega_o$ using Integrated Directional Encoding (IDE). Pass the Gaussian BRDF features $\rho_i$, the queried illumination $L_i$, and the encoded view direction $\omega_o$ through the neural network $\mathcal{F}$ to obtain the specular color $\mathbf{c}_{s,i}$:
    $$ \mathcal{F} : \{ \rho_i, L_i, \omega_o \} \mapsto \mathbf{c}_{s,i} $$
    [S1].
5.  **Compute Outgoing Radiance:** Combine the diffuse color $\mathbf{c}_{d,i}$ and the computed specular color $\mathbf{c}_{s,i}$:
    $$ \mathbf{c}_i(\omega_o) = \mathbf{c}_{d,i} + \mathbf{c}_{s,i}(\omega_o) $$
    [S1].
6.  **Rasterization:** Render the scene using the alpha-blending rasterization pipeline of 3D Gaussian Splatting [S1]. For a pixel $\mathbf{u}$, the color $\hat{\mathbf{C}}$ is:
    $$ \hat{\mathbf{C}} = \sum_{i \in N} T_i g_i(\mathbf{u} | \mathbf{\mu}', \mathbf{\Sigma}') \alpha_i \mathbf{c}_i, \quad T_i = \prod_{j=1}^{i-1} (1 - g_j(\mathbf{u} | \mathbf{\mu}', \mathbf{\Sigma}') \alpha_j) $$
    where $g_i$ is the 2D Gaussian function centered at $\mathbf{\mu}'$ with covariance $\mathbf{\Sigma}'$ [S1].

**Optimization Strategy:**
1.  **Phase 1 (Iterations 0–3,000):** Train the model using only the diffuse color $\mathbf{c}_d$. Optimize Gaussian means, opacity, covariance, and diffuse color. The illumination grid is also optimized [S1].
2.  **Phase 2 (Iterations 3,000–30,000):** Add specular colors to the Gaussians. Continue optimizing all parameters, including BRDF features $\rho_i$ and the illumination grid [S1].
3.  **Grid Boundary Management:** Midway through training, shrink the illumination grid boundary to fit the Gaussians and resample the grid with the same number of voxels to maintain resolution [S1].
4.  **Loss Function:** Minimize the combined L1 and D-SSIM loss:
    $$ \mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda \mathcal{L}_{\text{D-SSIM}} $$
    with $\lambda = 0.2$ [S1].

**Parameters:**
*   Illumination grid resolution: $150^3$ voxels [S1].
*   Training iterations: 30,000 [S1].
*   Loss weight $\lambda$: 0.2 [S1].
*   BRDF features: 16x3 SH coefficients + 4 parameters (base color, roughness) [S1].

**Edge Cases and Pitfalls:**
*   **Grid Boundary:** If the illumination grid boundary does not encompass the Gaussians, illumination queries may be invalid or inaccurate. The midway shrinkage strategy mitigates this by adapting the grid to the Gaussian distribution [S1].
*   **Specular Stability:** Adding specular colors too early can destabilize training. The two-phase training strategy (diffuse first, then specular) is critical for convergence [S1].
*   **Memory:** While compact, the factorized tensor grid still consumes memory. However, it uses 87.5% less voxels compared to TensoRF on synthetic datasets [S1].

## Operating characteristics
*   **Time Complexity:** Rendering speed is comparable to 3DGS because the illumination field is pre-calculated/stored in volumes and evaluated via fast interpolation, avoiding per-ray tracing [S1]. Training involves optimizing the grid and neural network, which adds overhead compared to vanilla 3DGS but remains efficient due to the compact tensor representation [S1].
*   **Memory Complexity:** The illumination grid uses a coarse resolution ($150^3$) and factorized representation, resulting in 87.5% less memory usage than TensoRF on synthetic datasets [S1]. The memory footprint is low relative to the number of optimized Gaussians [S1].
*   **Performance:** On the Shiny Blender dataset, 3iGS achieves an average PSNR of 27.51, SSIM of 0.930, and LPIPS of 0.045 [S1]. It outperforms 3DGS and GaussianShader in scenes with reflective surfaces [S1].
*   **Strengths:** Accurate modeling of specular view-dependent effects; fast rendering; compact illumination representation.
*   **Limitations:** Requires two-phase training; relies on a neural network for shading which adds slight complexity to the pipeline compared to pure SH [S1].

## Background
3iGS was introduced by Z.J. Tang and T.J. Cham [S1]. It addresses the limitations of 3D Gaussian Splatting, which optimizes outgoing radiance in isolation using spherical harmonics, leading to poor view-dependent specular effects [S1]. The method is inspired by conventional computer graphics engines that use baked illumination fields for fast rendering [S1]. It differs from GaussianShader, which uses global environment lights and SH-based color, by using local grid-based tensors and BRDF features [S1].

## Sources
[S1] 3iGS: Factorised Tensorial Illumination for 3D Gaussian Splatting, https://core.ac.uk/works/391741134, fetched 2026-10-29
