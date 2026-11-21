# Gaussian Avatars

Also called: FLAME Gaussian Splatting, PSAvatar, PSAvatar / Relightable Avatar, Relightable Avatar

## Overview
Gaussian Avatars is a method for reconstructing photorealistic, animatable head avatars by associating 3D Gaussians with the triangles of a FLAME parametric head model [S1][S3, unofficial]. It solves the challenge of dynamic avatar reconstruction by constraining each Gaussian to remain within its corresponding mesh triangle, enabling riggable animation driven by pose and expression parameters [S1][S3, unofficial]. It is distinguished from sibling methods like Gaussian Head Avatar (which uses MLPs for position prediction) and Gaussian Avatar (which uses SMPL for full-body) by its explicit triangle-constrained initialization and optimization strategy for bust-level reconstruction [S1][S3, unofficial].

## How it works
**Inputs:** Multi-view video of a subject's head; initial FLAME mesh fit (e.g., via MICA or DECA) [S2, unofficial].
**Outputs:** A set of 3D Gaussians, each associated with a FLAME mesh triangle, with parameters optimized for appearance and deformation.

**Data Structures:**
*   **FLAME Model:** Parametric head mesh with $N$ triangles. Each triangle $t_i$ has vertices $v_{i,1}, v_{i,2}, v_{i,3}$ in local space.
*   **Gaussian Set:** A list of $N$ Gaussians, one per triangle.
    *   **Base Position:** The centroid of the triangle in the rest pose.
    *   **Deformation Offsets:** Learnable parameters (or network outputs) that define the offset of the Gaussian center from the deformed triangle centroid, conditioned on pose/expression.
    *   **Covariance:** $3 \times 3$ matrix (or 6-vector for symmetric positive definite representation).
    *   **Opacity:** Scalar $\alpha \in [0, 1]$.
    *   **Spherical Harmonics (SH):** Coefficients for view-dependent color.

**Steps:**
1.  **Initialization:** Fit a FLAME model to the input video to obtain a rest-pose mesh [S2, unofficial]. Initialize one Gaussian per triangle. The Gaussian's initial position is the triangle centroid. The Gaussian is constrained to lie within the triangle's barycentric coordinates [S1][S3, unofficial].
2.  **Pose-Conditioned Deformation:** For a given pose $\theta$ and expression $\beta$, the FLAME model deforms the mesh vertices to $v'_{i,j}$. The triangle centroid moves to $c'$. The Gaussian position $p$ is computed as $p = c' + \delta$, where $\delta$ is a deformation offset. The offset $\delta$ is either a learnable parameter per Gaussian or predicted by a small network conditioned on $\theta, \beta$ [S1][S2, unofficial].
3.  **Constrained Optimization:** The Gaussian parameters (position offsets, scale, rotation, SH coefficients, opacity) are optimized by minimizing the photometric loss between the rendered image and the input video frame [S1][S3, unofficial]. The constraint that the Gaussian remains within the triangle is enforced during optimization, often by parameterizing the position in barycentric coordinates or using a projection step [S1][S3, unofficial].
4.  **Rendering:** The Gaussians are splatted onto the image plane using standard differentiable Gaussian splatting [S1][S4, unofficial]. The rendering is differentiable with respect to the Gaussian parameters and deformation offsets.

**Formulas:**
*   Triangle centroid in rest pose: $c = \frac{1}{3}(v_1 + v_2 + v_3)$.
*   Deformed centroid: $c' = \text{FLAME}(v_1, v_2, v_3, \theta, \beta)$.
*   Gaussian position: $p = c' + \delta(\theta, \beta)$, where $\delta$ is the learned offset [S1][S2, unofficial].
*   Barycentric constraint: If $p$ is represented in barycentric coordinates $(u, v, w)$ relative to the deformed triangle, then $u, v, w \ge 0$ and $u+v+w=1$ [S1][S3, unofficial].

**Parameters:**
*   Learning rate: Not given in the sources.
*   Number of Gaussians: Equal to the number of triangles in the FLAME mesh [S1][S3, unofficial].

**Edge Cases:**
*   **Degenerate Triangles:** Triangles with zero area in the rest pose may cause numerical instability. [G] Initialize such Gaussians with small random offsets or skip them.
*   **Precision:** Barycentric coordinate constraints must be handled carefully to avoid drift outside the triangle during optimization. [G] Use a softmax or projection layer to enforce $u+v+w=1$ and non-negativity.

## Subtopics

### Relightable Gaussian Avatars
Extends Gaussian Avatars by adding a relightable BRDF to the Gaussians, allowing for dynamic lighting changes [S2, unofficial].

### PSAvatar
A variant that uses a FLAME + light rig for photoreal head reconstruction with relighting capabilities [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of triangles and the complexity of the deformation network. Not given in the sources.
*   **Memory Complexity:** Proportional to the number of triangles in the FLAME mesh. Each Gaussian stores position, covariance, opacity, and SH coefficients. Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** High photorealism for head avatars; explicit mesh constraint ensures anatomical plausibility; riggable with FLAME parameters [S1][S2, unofficial].
*   **Limitations:** Limited to bust/head region; requires initial FLAME fit; may struggle with extreme expressions if the triangle constraint is too rigid [S1][S3, unofficial].
*   **When to use:** When high-quality, animatable head avatars are needed from multi-view video.
*   **When not to use:** For full-body avatars (use Gaussian Avatar or 3DGS-Avatar) or when real-time inference with minimal training is required.

## Background
Gaussian Avatars was introduced as a method to address the challenges of dynamic avatar reconstruction using 3D Gaussian Splatting [S1][S3, unofficial]. It leverages the FLAME parametric head model as a prior to constrain the Gaussians to the head surface, enabling riggable animation [S1][S3, unofficial]. The method is part of the broader family of Gaussian Avatar Reconstruction techniques [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
