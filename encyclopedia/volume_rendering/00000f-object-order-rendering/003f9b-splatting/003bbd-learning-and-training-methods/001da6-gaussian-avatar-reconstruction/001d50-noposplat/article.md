# NoPoSplat

Also called: No-Poses Splatting

## Overview
NoPoSplat is a 3D Gaussian Splatting method designed for novel view synthesis from sparse or unposed input images [S1][S2, unofficial]. It eliminates the dependency on known camera poses by establishing a canonical coordinate system anchored to a single input view [S1][S2, unofficial]. By introducing intrinsics token embeddings, the method resolves scale ambiguity, achieving superior reconstruction quality in scenarios with minimal view overlap compared to methods requiring precise pose initialization [S1][S2, unofficial].

## How it works
NoPoSplat operates by jointly optimizing Gaussian parameters and camera intrinsics within a pose-free framework [S1][S2, unofficial]. Unlike standard Gaussian Splatting which assumes fixed or known extrinsics, NoPoSplat treats the camera pose as implicit by anchoring the scene to a reference view [S1][S2, unofficial].

**Inputs and Outputs:**
*   **Inputs:** A set of $N$ input images $\{I_i\}_{i=1}^N$. No camera poses are required.
*   **Outputs:** A set of 3D Gaussians $\mathcal{G} = \{g_j\}_{j=1}^M$ defined in a canonical coordinate system, and estimated camera intrinsics.

**Data Structures:**
*   **Gaussian Parameters:** Each Gaussian $g_j$ is defined by position $\mathbf{p}_j \in \mathbb{R}^3$, covariance matrix $\mathbf{\Sigma}_j \in \mathbb{R}^{3 \times 3}$ (or its rotation $\mathbf{R}_j$ and scale $\mathbf{s}_j$), opacity $\alpha_j \in [0, 1]$, and spherical harmonic coefficients for color $\mathbf{c}_j$ [S1][S2, unofficial].
*   **Intrinsics Tokens:** Learnable embeddings representing camera intrinsic parameters (focal length, principal point) [S1][S2, unofficial].

**Steps:**
1.  **Reference Frame Selection:** Select one input image as the reference view. The canonical coordinate system is defined relative to this view [S1][S2, unofficial].
2.  **Intrinsics Embedding:** Initialize learnable tokens for camera intrinsics. These tokens are used to resolve the scale ambiguity inherent in monocular or sparse-view reconstruction [S1][S2, unofficial].
3.  **Gaussian Initialization:** Initialize 3D Gaussians in the canonical space. The exact initialization strategy (e.g., random, depth-based) is not specified in the provided sources [G].
4.  **Differentiable Rendering:** Render the current set of Gaussians to generate images for all input views. The rendering process uses the estimated intrinsics and the fixed canonical positions [S1][S2, unofficial].
5.  **Loss Computation:** Compute the photometric loss (e.g., $\mathcal{L}_{L1}$) between rendered images and input images [S1][S2, unofficial].
6.  **Optimization:** Update Gaussian parameters and intrinsics tokens via gradient descent to minimize the loss [S1][S2, unofficial].

**Formulas:**
The rendering equation remains the standard alpha-blending of Gaussians [S1][S2, unofficial]:
$$ C = \sum_{i \in \mathcal{N}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
where $C$ is the final color, $\mathcal{N}$ is the set of sorted Gaussians, $c_i$ is the color, and $\alpha_i$ is the opacity [S1][S2, unofficial].

**Parameters:**
*   Number of Gaussians: Not given in the sources.
*   Learning rate: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** Without known poses, the absolute scale of the scene is ambiguous. The intrinsics token embeddings are critical to resolve this [S1][S2, unofficial].
*   **Minimal View Overlap:** The method is specifically noted to perform well in scenarios with minimal view overlap, where traditional pose estimation might fail [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Achieves superior novel view synthesis quality, especially in scenarios with minimal view overlap [S1][S2, unofficial].
*   **Strengths:** Eliminates the need for known camera poses; robust to sparse inputs.
*   **Limitations:** Requires optimization of intrinsics; potential for scale ambiguity if not properly constrained.

## Background
NoPoSplat was introduced to address critical limitations in existing 3D Gaussian Splatting methods that rely on known camera poses [S1][S2, unofficial]. It builds on the principles of Gaussian Splatting [946-gaussian-splatting.md] but adapts them for pose-free reconstruction [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
