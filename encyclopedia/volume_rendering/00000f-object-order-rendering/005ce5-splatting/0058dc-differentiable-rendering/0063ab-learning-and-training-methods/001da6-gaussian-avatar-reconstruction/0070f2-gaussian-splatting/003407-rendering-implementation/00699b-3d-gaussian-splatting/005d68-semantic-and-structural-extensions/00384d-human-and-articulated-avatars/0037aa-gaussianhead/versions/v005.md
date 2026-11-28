# GaussianHead

Also called: GaussianHead [S1]

## Overview
GaussianHead is a method for animatable head reconstruction that uses a tri-plane representation and motion fields to simulate geometrically changing heads in continuous movements [S1][S2, unofficial]. It is designed to render rich textures, including skin and hair [S1][S2, unofficial]. It differs from siblings like GART and D3GA by explicitly leveraging tri-plane features for deformation rather than relying solely on skeleton-based or mesh-based deformation priors.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians initialized in a canonical space, pose parameters $\theta$, and expression coefficients.
*   **Outputs:** A rendered image $\mathbf{C}(p)$ for each pixel $p$.

**Data Structures:**
*   **Tri-plane Representation:** A set of three 2D feature planes (XY, YZ, XZ) storing latent features for the head geometry and appearance [S1][S2, unofficial].
*   **Motion Fields:** A learned field that predicts the displacement of Gaussians based on pose and expression [S1][S2, unofficial].

**Steps:**
1.  **Canonical Initialization:** Initialize 3D Gaussians in a neutral reference pose (e.g., T-pose or neutral expression) [G].
2.  **Tri-plane Sampling:** For each Gaussian, sample latent features from the tri-plane representation based on its canonical position [S1][S2, unofficial].
3.  **Deformation Prediction:** Use the motion fields to predict the deformation of the Gaussians based on the current pose $\theta$ and expression [S1][S2, unofficial]. This maps canonical positions to target positions:
    $$
    \mathbf{p}_i^{target} = D(\mathbf{p}_i^{canonical}, \theta, \text{motion\_field})
    $$
    The rotation and scale may also be transformed to maintain physical consistency [G].
4.  **Rendering:** Render the deformed Gaussians using the standard differentiable splatting pipeline described in [Human and Articulated Avatars](8209-human-and-articulated-avatars.md).

**Parameters:**
*   Tri-plane resolution: Not given in the sources.
*   Motion field network depth: Not given in the sources.

**Edge Cases and Pitfalls:**
*   Precision issues in tri-plane sampling may occur if the Gaussian positions fall outside the normalized bounding box of the tri-planes [G].
*   Degenerate expressions (e.g., extreme mouth openings) may cause self-intersections if the motion field is not sufficiently regularized [G].

## Subtopics

### FlashAvatar
FlashAvatar embeds a uniform 3D Gaussian field in a parametric face model and learns additional spatial offsets to capture facial details [S2, unofficial]. It pushes the rendering speed to 300 FPS [S2, unofficial]. It is chosen when high-speed rendering is prioritized over the explicit tri-plane motion field approach of GaussianHead.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** FlashAvatar, a variant, achieves 300 FPS [S2, unofficial].
*   **Strengths:** Capable of rendering rich textures including skin and hair [S1][S2, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** When reconstructing animatable heads with continuous movements and rich textures [S1][S2, unofficial].
*   **When not:** When real-time performance is not critical and simpler deformation models suffice.

## Background
GaussianHead was introduced in the context of recent advances in 3D Gaussian splatting for human and articulated avatars [S1][S2, unofficial]. It builds on the tri-plane representation and motion fields to handle geometric changes in head avatars [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
