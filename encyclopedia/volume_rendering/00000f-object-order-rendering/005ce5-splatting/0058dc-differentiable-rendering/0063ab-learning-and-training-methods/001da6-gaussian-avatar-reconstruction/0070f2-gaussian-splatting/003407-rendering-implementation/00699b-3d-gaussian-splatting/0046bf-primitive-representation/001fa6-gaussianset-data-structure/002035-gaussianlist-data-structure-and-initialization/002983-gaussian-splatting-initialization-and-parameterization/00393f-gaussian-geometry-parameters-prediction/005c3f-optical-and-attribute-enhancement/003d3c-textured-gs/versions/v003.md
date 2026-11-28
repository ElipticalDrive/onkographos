# Textured-GS

Also called: Gaussian Splatting with Spatially Defined Color and Opacity

## Overview
Textured-GS enhances the textural representation of individual 3D Gaussians by modeling spatially defined color and opacity variations across the Gaussian ellipsoidal surface using Spherical Harmonics (SH), without increasing the number of Gaussians or adding extra parameters beyond the existing SH framework [S1]. Unlike standard 3D Gaussian Splatting (3DGS) where each Gaussian retains a fixed opacity and shows a single color when viewed from a specific direction, Textured-GS enables color variation across the Gaussian ellipsoidal surface and opacity variations, allowing for deviations from the standard ellipsoidal shape such as 'unidirectional Gaussians' that are opaque on one side and transparent on the other [S1]. It differs from its sibling [Attribute-Conditioned SH Enhancement](15410-attribute-conditioned-sh-enhancement.md) by modifying the parameterization scheme to use the surface intersection vector for SH evaluation rather than the viewing vector, thereby enabling spatial variation within a single primitive [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians initialized from Mini-Splatting output [S1]. View-dependent color is not loaded as input [S1].
*   **Output:** Rendered images with spatially varying color and opacity per Gaussian.

**Data Structures:**
The method retains the standard 3DGS Gaussian structure but modifies the SH coefficient storage:
*   **Position:** $x \in \mathbb{R}^3$
*   **Rotation:** $q \in \mathbb{R}^4$ (Quaternion)
*   **Scale:** $s \in \mathbb{R}^3$ (Optimized as $\sqrt{s}$ for the ellipsoid equation, but actual Gaussian size in splatting remains defined by $s$) [S1].
*   **Opacity:** $\alpha \in [0, 1]$ (Base opacity, though effective opacity varies spatially) [S1].
*   **SH Coefficients:** 16 coefficients per Gaussian. Standard 3DGS uses 3 channels (RGB). Textured-GS uses 4 channels: 3 for color texture ($\gamma_{lm}$) and 1 for opacity texture ($\alpha_{lm}$) [S1].
*   **SH Degree Level:** Set to 3 for both color and opacity [S1].

**Algorithm Steps:**

1.  **Initialization:** Start with Gaussians from Mini-Splatting output [S1]. Do not load view-dependent color as input [S1]. Remove adaptive control modules (splitting/cloning) from the 3DGS framework to streamline optimization [S1].
2.  **Ray-Ellipsoid Intersection:** For a viewing ray $r(t) = o + td$ (where $o$ and $d$ are the ray’s origin and direction transformed into the ellipsoid’s local coordinate system), calculate the intersection points with the Gaussian ellipsoid defined by:
    $$ \frac{x^2}{s_x^2} + \frac{y^2}{s_y^2} + \frac{z^2}{s_z^2} = 1 $$
    Substituting the ray equation yields the quadratic equation $At^2 + Bt + C = 0$, where:
    $$ A = \frac{d_x^2}{s_x^2} + \frac{d_y^2}{s_y^2} + \frac{d_z^2}{s_z^2} $$
    $$ B = 2\left(\frac{o_x d_x}{s_x^2} + \frac{o_y d_y}{s_y^2} + \frac{o_z d_z}{s_z^2}\right) $$
    $$ C = \frac{o_x^2}{s_x^2} + \frac{o_y^2}{s_y^2} + \frac{o_z^2}{s_z^2} - 1 $$
    Solve for $t$ using the quadratic formula:
    $$ t = \frac{-B \pm \sqrt{B^2 - 4AC}}{2A} $$
    Select the intersection point with the lower $t$ value (closer to ray origin) [S1].
3.  **Normalization:** Transform the selected intersection point into the Gaussian's local coordinate system and normalize each dimension based on the scale $s$ to obtain the normalized vector $n$:
    $$ n = \left( \frac{x_{int}}{s_x}, \frac{y_{int}}{s_y}, \frac{z_{int}}{s_z} \right) $$
    This vector $n$ represents the position on the unit sphere corresponding to the intersection point on the ellipsoid [S1].
4.  **SH Evaluation (Color and Opacity):** Evaluate the SH basis functions $Y_{lm}$ using the normalized vector $n$ instead of the viewing vector $v$. Compute the color $c_i$ and opacity $\alpha_i$ for the Gaussian at this specific surface point:
    $$ c_i = \sum_{l=0}^{L} \sum_{m=-l}^{l} \gamma_{lm} Y_{lm}(n) $$
    $$ \alpha_i = \sum_{l=0}^{L} \sum_{m=-l}^{l} \alpha_{lm} Y_{lm}(n) $$
    Apply a Sigmoid activation function to both color ($\gamma$) and opacity ($\alpha$) coefficients to ensure smooth gradients [S1].
5.  **Rendering:** Composite the textured Gaussians using standard alpha blending, where the opacity $\alpha_i$ varies spatially across the Gaussian surface [S1].
6.  **Optimization:** Update SH coefficients incrementally after each pixel is processed, rather than aggregating loss at the end of the iteration. This is necessary because the parameterization vector $n$ varies with each pixel, unlike the constant view vector in standard 3DGS [S1].

**Parameters:**
*   **SH Degree Level:** 3 [S1].
*   **Loss Function Weight Lambda ($\lambda$):** 0.2 [S1].
*   **Learning Rate:** 0.0025 for color texture, 0.005 for opacity texture [S1].
*   **Activation Function:** Sigmoid for both color and opacity coefficients [S1].

**Pitfalls and Edge Cases:**
*   **Storage:** Requires 4 channels of SH coefficients (3 for color, 1 for opacity) instead of 3, increasing storage size slightly compared to baseline with the same Gaussian count [S1].
*   **Scale Parameterization:** Use the square root of the scale parameters ($\sqrt{s}$) for the ellipsoid equation during optimization to yield better performance for larger ellipsoids, though this does not alter the actual Gaussian size in splatting [S1].
*   **Adaptive Control:** The method removes adaptive control modules (splitting/cloning). Implementers must ensure these modules are disabled to maintain the intended optimization behavior [S1].

## Subtopics

### Recolorable Textures with Palette Color
Extends textured Gaussians with a learnable palette color $c_{pal}$ to enable real-time color editing by shifting the view-independent color [S2]. The view-independent color $c_{ind}$ of the $i$-th Gaussian at intersection point $p_i$ is defined as $c_{ind} = c_{pal} + (c_{tex})_i(p_i)$, where $c_{pal}$ is a learnable palette color parameter shared across all Gaussians in a basic scene, and $(c_{tex})_i(p_i)$ is the sampled texture color [S2]. $c_{pal}$ is initialized as the mean color of input multi-view images and is optimized using an RGB reconstruction loss combined with L1 sparsity regularization on $c_{tex}$ [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Uses less training memory than baseline methods with comparable Gaussian counts. On the Bicycle Scene (Mip-NeRF 360), it uses 4.53GB memory compared to 5.46GB for Mini-Splatting [S1]. However, it has a slightly larger file size (164MB vs 132MB for Mini-Splatting) due to the 4-channel SH storage [S1].
*   **Speed:** Rendering speed is comparable to Mini-Splatting. On the Bicycle Scene, it achieves 161 FPS [S1]. Training time is 10m52s on the Bicycle Scene [S1].

**Measured Performance:**
*   **Mip-NeRF 360:** SSIM 0.828, PSNR 28.54, LPIPS 0.189 with 0.49M Gaussians [S1]. Outperforms standard 3DGS (SSIM 0.815, PSNR 27.69) and Mini-Splatting (SSIM 0.822, PSNR 25.59) [S1].
*   **Tanks&Temples:** SSIM 0.841, PSNR 23.14, LPIPS 0.183 with 0.20M Gaussians [S1].
*   **Deep Blending:** SSIM 0.903, PSNR 29.41, LPIPS 0.243 with 0.35M Gaussians [S1].

**Strengths and Limitations:**
*   **Strengths:** Achieves comparable or superior rendering quality to standard 3DGS and Mini-Splatting while requiring significantly fewer Gaussians. Effectively addresses sharp feature representation and serrated edges seen in low-count Gaussian methods [S1].
*   **Limitations:** Requires incremental gradient updates which may complicate implementation compared to batch updates. The removal of adaptive control modules means the method relies on the quality of the initial Gaussian distribution (e.g., from Mini-Splatting) and cannot dynamically adjust density [S1].

## Background
Textured-GS was introduced to address the bottleneck of Gaussian representation in rendering performance, where standard Gaussians exhibit only a single color from a specific viewing angle [S1]. The method adapts the Mini-Splatting architecture to integrate textured Gaussians, validating the approach on pre-optimized 3D Gaussian scenes [S1]. It leverages the differentiable rendering process of 3D Gaussian Splatting [11] but modifies the SH evaluation to use spatial surface vectors [S1].

## Sources
[S1] Textured-GS: Gaussian Splatting with Spatially Defined Color and Opacity, https://core.ac.uk/works/391755422, fetched 2026-10-30
[S2] TexGS-VolVis: Expressive Scene Editing for Volume Visualization via Textured Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3634643, fetched 2026-10-30
