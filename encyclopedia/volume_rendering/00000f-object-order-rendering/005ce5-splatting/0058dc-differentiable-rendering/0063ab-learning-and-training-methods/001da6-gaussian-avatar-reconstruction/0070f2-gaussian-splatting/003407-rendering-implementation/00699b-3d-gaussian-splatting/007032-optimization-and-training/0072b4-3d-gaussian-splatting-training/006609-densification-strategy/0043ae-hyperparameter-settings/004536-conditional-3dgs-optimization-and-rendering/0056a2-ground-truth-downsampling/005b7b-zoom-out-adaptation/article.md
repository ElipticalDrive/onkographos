# Zoom-out Adaptation

Also called: Downsample-based adaptation, Zoom-out pseudo-GT generation

## Overview
Zoom-out Adaptation is a post-training optimization technique that adapts a pre-trained 3D Gaussian Splatting (3DGS) model to render anti-aliased images at lower resolutions (zoom-out). It solves the problem of geometric dilation and aliasing artifacts (e.g., thick spokes on wheels) that occur when a high-resolution model is viewed at a distance or downsampled. Unlike [Mip-Splatting](15762-mip-splatting.md), which uses filters to mitigate dilation, this method generates pseudo-ground truth (pseudo-GT) by downsampling high-resolution renderings and re-optimizes the Gaussians to fit this multi-scale supervision, achieving a 5dB PSNR improvement over baseline 3DGS on zoom-out tasks [S1].

## How it works
**Inputs:** A pre-trained 3DGS model (positions, covariances, opacities, colors) trained on full-resolution ($\times 1$) images; camera poses for the target scene.
**Outputs:** An adapted 3DGS model optimized for low-resolution rendering.

**Data Structures:**
The 3DGS model retains the standard structure: a list of $N$ Gaussians, each defined by mean position $\mu \in \mathbb{R}^3$, covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ (or quaternion/scale), opacity $\alpha \in [0,1]$, and spherical harmonics coefficients for color.

**Steps:**
1.  **Baseline Training:** Train the initial 3DGS model on full-resolution ($\times 1$) training images until convergence [S1].
2.  **High-Resolution Rendering:** Render the scene using the trained model at the target lower resolutions (e.g., $\times 1/2$, $\times 1/4$, $\times 1/8$ of the original training resolution) [S1]. Let $I_{HR}$ be the rendered image at the original resolution.
3.  **Pseudo-GT Generation:** Downsample the high-resolution renderings to create the pseudo-ground truth $I_{pseudo}$ for the target resolution. This is typically done using standard image downsampling (e.g., average pooling or bilinear filtering) [S1].
    *   Formula: $I_{pseudo} = \text{Downsample}(I_{HR}, \text{scale})$
4.  **Adaptation Optimization:** Optimize the 3D Gaussian parameters (positions, scales, opacities, colors) by minimizing the reconstruction loss between the rendered image at the target low resolution and the $I_{pseudo}$ [S1].
    *   Loss: $\mathcal{L} = \mathcal{L}_{rec}(I_{rendered\_LR}, I_{pseudo}) + \lambda \mathcal{L}_{reg}$
    *   The optimization uses the same camera poses as the original training but renders at the lower resolution [S1].
5.  **Convergence:** Run optimization until convergence. For zoom-out, this typically takes 0.5K iterations [S1].

**Parameters:**
*   **Target Resolutions:** $\times 1/2$, $\times 1/4$, $\times 1/8$ [S1].
*   **Optimization Views:** Test views or synthetic views sampled from the estimated camera trajectory are preferred over training views to retain novel-view performance [S1].
*   **Iterations:** 0.5K iterations for zoom-out [S1].

**Edge Cases and Pitfalls:**
*   **Pseudo-GT Quality:** The quality of guidance varies by dataset. For the Blender dataset (isolated objects), supervision is intensive and high-quality. For Mip-NeRF 360 (complex backgrounds), the pseudo-GT has limited quality and provides weaker guidance [S1].
*   **Needle-like Aliasing:** Without adaptation, 3DGS exhibits needle-like aliasing due to intrinsic shrinkage bias when zoomed in, but for zoom-out, the primary artifact is dilation [S1].

## Subtopics

### Zoom-in Adaptation
A technique to adapt a pre-trained 3DGS model for high-resolution rendering (zoom-in) by using a super-resolution network (SwinIR) to generate high-resolution pseudo-GT from low-resolution training data. Unlike zoom-out, which downsamples renderings, this method upscales supervision. It converges in 1K iterations and generates novel views with high fidelity, avoiding the needle-like aliasing of 3DGS and the high-frequency trade-offs of Mip-Splatting [S1].

## Operating characteristics
*   **Time Complexity:** Convergence is very fast. Zoom-out adaptation converges within 0.5K iterations (on the order of seconds) [S1]. This is faster than the zoom-in adaptation (1K iterations) and significantly faster than Mip-Splatting's filter update, which is not compatible with all 3DGS variations [S1].
*   **Memory Complexity:** No significant additional memory overhead during rendering, as it reuses the standard 3DGS structure. The optimization phase requires storing the pseudo-GT images [S1].
*   **Performance:** Improves PSNR by 5dB compared to baseline 3DGS on zoom-out tasks [S1]. Visual results show superior handling of dilation artifacts (e.g., wheel spokes) compared to Mip-Splatting [S1].
*   **Strengths:** Plug-in compatibility with dynamic 3DGS models; no disturbance to deformation networks [S1]. Reduces redundancy by allowing up to 25% reduction in splatted primitives without performance loss [S1].
*   **Limitations:** Effectiveness depends on the quality of pseudo-GT, which is lower for complex scenes like Mip-NeRF 360 compared to synthetic datasets like Blender [S1].

## Background
Originated in "Mipmap-GS: Let Gaussians Deform with Scale-Specific Mipmap for Anti-Aliasing Rendering" (2025) [S1]. The method addresses the limitation of standard 3DGS, which is optimized for a specific resolution and suffers from aliasing when viewed at different scales. It builds on the observation that downsampling high-quality renderings preserves essential scene details, providing a robust supervision signal for scale-specific adaptation [S1].

## Sources
[S1] Mipmap-GS: Let Gaussians Deform with Scale-Specific Mipmap for Anti-Aliasing Rendering, https://doi.org/10.1109/3dv66043.2025.00091, fetched 2026-10-27
