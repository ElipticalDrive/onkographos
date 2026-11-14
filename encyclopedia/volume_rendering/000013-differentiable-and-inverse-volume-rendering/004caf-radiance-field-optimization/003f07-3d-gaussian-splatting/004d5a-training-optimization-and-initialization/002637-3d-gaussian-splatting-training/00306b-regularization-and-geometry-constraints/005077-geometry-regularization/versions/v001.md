# Geometry Regularization

Also called: Gaussian Flattening, Normal Consistency and Flattening

## Overview
Geometry regularization improves the geometric fidelity and smoothness of 3D Gaussian Splatting scenes by constraining the shape and orientation of the Gaussian kernels, distinct from depth supervision which only constrains their spatial location [S1, S2, unofficial]. It sets itself apart from sibling methods like [Depth Regularization](7616-depth-regularization.md) by enforcing structural constraints (flattening and normal alignment) rather than positional errors, and from [Frequency Regularization](6130-frequency-regularization.md) by operating in the spatial domain to reduce floaters and improve surface continuity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Per-Gaussian parameters (centers $\mu$, scales $s$, rotations $q$, opacities $\alpha$), rendered depth map $D$, and optionally a reference point cloud for scale refitting [S1, unofficial].
*   **Outputs:** A scalar regularization loss $\mathcal{L}_{reg}$ added to the total loss [Parent].

**Data Structures:**
*   **Gaussian Scales:** Vector $s \in \mathbb{R}^3$ representing the scaling factors along the three principal axes of each Gaussian [S2, unofficial].
*   **Normals:** Per-Gaussian normals $n_g$ derived from the rotation quaternion $q$ (specifically the third column of the rotation matrix) [S1, S2, unofficial]. Rendered normals $n_r$ derived from the gradient of the rendered depth map [S1, unofficial].

**Steps:**
1.  **Enablement:** The method is controlled by a boolean parameter `use_normal_loss` in `OptimizationParams` [S1, unofficial].
2.  **Depth Refitting (Optional but recommended):** If using external depth maps, refit them against a prior point cloud to ensure scale consistency. Discard depth maps that fail refitting [S1, unofficial].
3.  **Flattening Loss:** Minimize the smallest component of the scaling factor $s$ towards zero. This forces Gaussians to flatten into 3D "surfels" aligned with the surface, aiding normal determination by selecting the shortest scaled edge [S2, unofficial].
4.  **Normal Consistency Loss:** Compute normals from the rendered depth map ($n_r$) and compare them with the per-Gaussian normals ($n_g$). Enforce alignment between $n_r$ and $n_g$ [S1, unofficial].

**Formulas:**
The total regularization loss is:
$$ \mathcal{L}_{reg} = \mathcal{L}_{flatten} + \mathcal{L}_{normal} $$

Where $\mathcal{L}_{flatten}$ minimizes the smallest scale component $s_{min}$:
$$ \mathcal{L}_{flatten} = s_{min} $$
Here, $s_{min} = \min(s_x, s_y, s_z)$ for each Gaussian [S2, unofficial].

$\mathcal{L}_{normal}$ enforces alignment between rendered and Gaussian normals (implementation detail: typically cosine similarity or angle loss) [S1, unofficial].

**Parameters:**
*   `use_normal_loss`: Boolean. Enables the geometry regularization pipeline [S1, unofficial].
*   $\lambda_{reg}$: Scalar weight for the regularization term in the total loss [Parent].

**Edge Cases and Pitfalls:**
*   **Scale Mismatch:** Depth maps must be refitted to the scene scale; otherwise, normal consistency may fail due to scale discrepancies [S1, unofficial].
*   **Sky/Background:** Geometry regularization may pull distant geometry onto itself if not handled. Using a "Sky Dome" (frozen shell of Gaussians) or hemisphere can mitigate floaters in sky regions [S1, unofficial].
*   **Floater Trade-off:** While flattening reduces floaters, it may reduce the number of remaining points, potentially compromising detail if not balanced with pruning mechanisms [S2, unofficial].

## Subtopics

### Scaling Loss for Geometry Modeling
A specific variant that focuses exclusively on flattening Gaussian kernels into 3D surfels by minimizing the smallest component of the scaling factor $s \in \mathbb{R}^3$ towards zero [S2, unofficial]. It aids in determining Gaussian normals by selecting the shortest scaled edge [S2, unofficial]. Choose this when surface alignment is the primary goal and normal consistency with rendered depth is handled separately or implicitly. It is a core component of the broader Geometry Regularization method [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Adds minimal overhead per training step, primarily involving scale extraction and normal computation from depth gradients [S1, S2, unofficial].
*   **Memory Complexity:** Negligible additional memory; uses existing Gaussian parameters and rendered buffers [S1, unofficial].
*   **Strengths:** Significantly reduces "floaters" (errant Gaussians in empty space) and improves surface smoothness and geometric coherence [S1, S2, unofficial].
*   **Limitations:** May reduce the number of active Gaussians, potentially losing fine detail if pruning is too aggressive [S2, unofficial]. Requires careful handling of background/sky regions to avoid pulling geometry inward [S1, unofficial].
*   **When to use:** When geometric accuracy and surface continuity are prioritized over raw PSNR, or when floaters are prevalent.
*   **When not to use:** When the scene is highly volumetric or non-surface-like, or when computational budget is extremely tight.

## Background
Originating from the need to improve the structural quality of 3D Gaussian Splatting beyond color reconstruction, this technique was formalized in implementations such as `GaussianSplatting.jl` and frameworks like `GauStudio` [S1, S2, unofficial]. It builds upon the observation that flattening Gaussians into surfels better represents surfaces than isotropic kernels [S2, unofficial]. The method is widely used in modern 3DGS pipelines to enhance geometry regularization alongside depth and normal supervision [S1, S2, unofficial].

## Sources
[S1] Better Gaussian Splatting in Julia, https://pxl-th.github.io/blog/better-gs-julia/, fetched 2026-10-26 (unofficial)
[S2] GauStudio: A Modular Framework for 3D Gaussian Splatting and Beyond, https://doi.org/10.48550/arXiv.2403.19632, fetched 2026-10-30 (unofficial)
