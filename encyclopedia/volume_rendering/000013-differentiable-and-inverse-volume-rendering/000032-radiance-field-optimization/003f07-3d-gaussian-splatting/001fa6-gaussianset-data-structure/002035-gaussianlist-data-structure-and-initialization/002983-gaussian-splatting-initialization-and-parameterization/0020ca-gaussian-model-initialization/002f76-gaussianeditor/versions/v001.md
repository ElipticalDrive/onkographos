# GaussianEditor


## Overview
GaussianEditor is a text-driven local editing method for 3D Gaussian Splatting that leverages 2D image-editing models to drive updates of 3D Gaussians [S1]. It solves the problem of localized appearance and geometry modification in 3D scenes by repeatedly rendering views, applying 2D edits, and back-propagating the differences to Gaussian parameters [S1]. It is distinguished from sibling methods like GScalpel (which uses paint-bucket/boolean logic) and Style-GS (which focuses on stylization) by its use of Hierarchical Gaussian Semantic Tracing and diffusion-guided refinement for precise text-driven localization [S2, unofficial].

## How it works
**Inputs:** A pre-trained 3D Gaussian Splatting model (set of Gaussians with position, covariance, opacity, SH coefficients), a text prompt describing the desired edit, and optionally a mask for local editing.
**Outputs:** An updated set of 3D Gaussians reflecting the edit.

**Data Structures:**
*   **Gaussian Set:** Standard 3DGS representation. Each Gaussian $i$ has:
    *   $\mu_i \in \mathbb{R}^3$: Mean position.
    *   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (often parameterized via rotation $R$ and scale $S$).
    *   $\alpha_i \in [0, 1]$: Opacity.
    *   $c_{i, l, m}$: Spherical Harmonics coefficients for view-dependent color.

**Steps:**
1.  **Hierarchical Gaussian Semantic Tracing:** Identify the subset of Gaussians corresponding to the text prompt. This involves lifting 2D semantic information (from a diffusion model or segmentation prior) to 3D to trace which Gaussians contribute to the target region [S2, unofficial].
2.  **Rendering:** Render the current 3D Gaussian scene to 2D images from multiple camera viewpoints [S1].
3.  **2D Edit Application:** Apply a 2D image-editing model (e.g., a text-conditioned diffusion editor) to the rendered images to generate target images that reflect the desired edit [S1].
4.  **Back-propagation:** Compute the difference between the edited 2D images and the original rendered images. Back-propagate this gradient through the differentiable rasterizer to update the Gaussian parameters ($\mu, \Sigma, \alpha, c$) [S1].
5.  **Diffusion-Guided Refinement:** Iteratively refine the Gaussian parameters using guidance from the diffusion model to ensure consistency and reduce artifacts [S2, unofficial].

**Parameters:**
*   Learning rate for Gaussian updates: Not given in the sources.
*   Number of iterations: Not given in the sources.

**Edge Cases:**
*   **Unintended Deformations:** Due to weak 3D geometry priors, edits may cause unintended deformations in non-target regions [S1].
*   **Computational Cost:** The pipeline is computationally heavy due to repeated rendering and diffusion model inference [S1].

## Operating characteristics
*   **Time Complexity:** High, due to repeated rendering and 2D diffusion model inference [S1].
*   **Memory Complexity:** Standard 3DGS memory plus overhead for 2D edit model [S1].
*   **Strengths:** Intuitive, compatible with powerful 2D priors, enables text- or mask-driven appearance and geometry changes [S1].
*   **Limitations:** Computationally heavy, can suffer from unintended deformations [S1].
*   **When to use:** When text-driven local editing is required and computational resources are available.
*   **When not to use:** When real-time performance is critical or when strong 3D geometry preservation is required without iterative refinement.

## Background
GaussianEditor was introduced by Chen et al. in CVPR'24 [S2, unofficial]. It belongs to the family of methods that leverage 2D image-editing models to drive updates of 3D Gaussians [S1]. Follow-up works have improved spatial localization and multi-view consistency via attention, view correspondence, or iterative refinement [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
