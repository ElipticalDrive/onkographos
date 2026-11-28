# 2D Gaussian Splatting

Also called: 2D splats, 2D-GS, 2DGS Scene Representation, Perspective accurate Gaussian Splatting, Surfel Splatting

## Overview
2D Gaussian Splatting (2DGS) is a surface reconstruction and novel view synthesis technique that replaces the volumetric 3D Gaussian ellipsoids of standard Gaussian Splatting with oriented, zero-thickness 2D disks (surfels) [S1, S4][S9, unofficial]. It solves the problem of ambiguous geometry and poor mesh extraction inherent in 3D Gaussian representations by enforcing primitives to align with local surface tangent planes, thereby providing unambiguous surface normals and improving multi-view depth consistency [S2, S9, unofficial]. Unlike its sibling [3D Gaussian Splatting](6068-3d-gaussian-splatting.md), which models scenes as volumetric blobs, 2DGS models surfaces directly, making it superior for high-resolution normal map recovery and clean mesh extraction [S4][S9, unofficial].

## How it works
**Inputs:** A set of $N$ input images $\{I_i\}$ with known camera poses $\{P_i\}$ (rotation and translation) [S2, unofficial].
**Outputs:** A scene representation consisting of a set of $M$ 2D Gaussian primitives, each defined by position, scale, rotation, opacity, and color [S2, unofficial][S8].

**Data Structures:**
Each primitive $j$ is defined by the tuple $G_j = \{\boldsymbol{\mu}_j, \mathbf{S}_j, \mathbf{R}_j, \alpha_j, \mathbf{c}_j\}$ [S2, unofficial][S8]:
*   $\boldsymbol{\mu}_j \in \mathbb{R}^3$: The 3D center position in world space [S2, unofficial][S8].
*   $\mathbf{S}_j \in \mathbb{R}^2$: Learnable scaling factors $(s_u, s_v)$ defining the extent of the disk along its local tangent axes [S2, unofficial].
*   $\mathbf{R}_j \in \mathbb{R}^{3 \times 3}$: Rotation matrix derived from a learnable quaternion $\mathbf{q}_j \in \mathbb{R}^4$ [S2, unofficial][S8]. The surface normal $\mathbf{n}_j$ is determined directly by $\mathbf{R}_j$ [S8].
*   $\alpha_j \in \mathbb{R}^1$: Learnable opacity parameter per splat [S2, unofficial].
*   $\mathbf{c}_j$: Color attributes (e.g., Spherical Harmonics coefficients) [S2, unofficial].

**Rendering Pipeline:**

1.  **Projection to Screen Space:**
    The 2D primitive is projected from world space to the image plane. Unlike 3D Gaussians which project a 3D covariance, 2DGS constructs a homography matrix $\mathbf{H}$ that maps local UV coordinates to screen space [S2, unofficial].
    
    The homography matrix $\mathbf{H}$ is defined as:
    $$
    \mathbf{H} = \begin{bmatrix} s_u & t_u & 0 & p_x \\ s_v & t_v & 0 & p_y \\ 0 & 0 & 0 & 1 \end{bmatrix}
    $$
    where:
    *   $s_u, s_v$ are the learnable scaling factors from $\mathbf{S}_j$ [S2, unofficial].
    *   $t_u, t_v$ are tangential vectors derived from the rotation $\mathbf{R}_j$ (or quaternion $\mathbf{q}_j$) [S2, unofficial].
    *   $p_x, p_y$ are the projected center coordinates of the splat in screen space [S2, unofficial].
    
    *Note: The sources describe $\mathbf{H}$ as mapping local coordinates to screen space. In implementation, this matrix is used to determine the intersection of a ray with the planar splat.*

2.  **Ray-Splat Intersection and Density Evaluation:**
    For a pixel at screen coordinate $\mathbf{x}$, the ray-splat intersection is computed. The density contribution is evaluated using a 2D Gaussian function based on the UV coordinates $(u(\mathbf{x}), v(\mathbf{x}))$ of the intersection point on the local tangent plane of the splat [S2, unofficial]:
    $$
    G(\mathbf{x}) = \exp \left( - \frac{u(\mathbf{x})^2 + v(\mathbf{x})^2}{2} \right)
    $$
    Here, $u(\mathbf{x})$ and $v(\mathbf{x})$ are the coordinates in the local tangent frame [S2, unofficial].

3.  **Opacity Adjustment:**
    The adjusted opacity $\alpha'_i$ for the splat at pixel $\mathbf{x}$ is calculated by combining the learnable opacity $\alpha_i$ with the spatial density $G(\mathbf{x})$ [S2, unofficial]:
    $$
    \alpha'_i = \alpha_i G(\mathbf{x})
    $$

4.  **Alpha Compositing:**
    Primitives are sorted by depth and blended using standard alpha compositing to produce the final color $C$ [S2, unofficial]:
    $$
    C = \sum_{i \in \mathcal{N}} \alpha'_i \mathbf{c}_i \prod_{j=1}^{i-1} (1 - \alpha'_j)
    $$
    where $\mathcal{N}$ is the set of splats contributing to the pixel [S2, unofficial].

**Parameters and Edge Cases:**
*   **Low-Pass Filtering:** A low-pass filter is employed during rendering or optimization to avoid degenerated line projections, which can occur when a 2D disk is viewed edge-on [S3].
*   **Degeneracy:** Because 2D Gaussians have zero thickness, they are prone to aliasing or visibility flickering if not properly filtered. The low-pass filter mitigates artifacts caused by high-frequency variations in the splat density [S3].
*   **Normal Consistency:** The normal $\mathbf{n}_j$ is fixed by the rotation $\mathbf{R}_j$. This provides a consistent normal for the primitive, which is critical for surface reconstruction but requires careful initialization to avoid flipping [S8][S9, unofficial].

## Subtopics

### [Flattened Gaussian Ellipsoid Rendering](1667-flattened-gaussian-ellipsoid-rendering.md)
Approximates 2D Gaussian ellipses by flattening 3D Gaussians within a differentiable volume rendering framework, enforcing a near-zero determinant covariance for stable surface-like rendering.

### [3D Surface Splatting](4524-3d-surface-splatting.md)
A differentiable surface splatting renderer for inverse rendering that uses oriented surfels and handles multiple surface layers per pixel through depth-interval merging, replacing binary depth tests with smooth compositing.

### [Spatially Varying Gaussian Splatting (SVGS)](8729-spatially-varying-gaussian-splatting-svgs.md)
Enhances 2DGS by defining color and opacity as spatially varying functions of the intersection point on the primitive, allowing a single primitive to fit complex textures.

### [Learning View-Dependent Splatting Kernels](9548-learning-view-dependent-splatting-kernels.md)
A framework that learns view-dependent 2D splatting kernels implicitly, improving reconstruction quality by allowing kernel profiles to vary with the camera view.

### [4D Scale-adaptive Filter](9868-4d-scale-adaptive-filter.md)
A filtering technique for dynamic 4D Gaussian Splatting that adaptively adjusts dilation based on primitive scale and visibility to prevent anisotropy distortion and eliminate invisible primitives.

## Operating characteristics
*   **Geometry Quality:** 2DGS significantly improves multi-view depth consistency and 3D reconstruction quality compared to 3DGS, particularly for hard surfaces and sharp edges [S2, unofficial][S4]. It enables the extraction of clean meshes and high-resolution normal maps [S7, S9, unofficial].
*   **Rendering Performance:** The rendering pipeline is similar to 3DGS (rasterization of splats), but the projection step involves computing the homography matrix and ray-splat intersection, which may add slight computational overhead per splat compared to the affine projection of 3D Gaussians [S2, unofficial].
*   **Limitations:** For pure image quality on fuzzy or non-surface scenes, 2DGS may perform slightly worse than 3DGS because the zero-thickness assumption does not model volumetric density well [S9, unofficial]. It is less suitable for scenes where the "surface" is ambiguous or volumetric [S9, unofficial].
*   **Use Cases:** Best suited for applications requiring accurate geometry, mesh extraction, and normal maps, such as inverse rendering, virtual production, and digital twins [S8][S9, unofficial].

## Background
2D Gaussian Splatting (2DGS) was introduced by Huang et al. in SIGGRAPH 2024 [S2, S6, S7, unofficial]. It was developed to address the limitations of 3D Gaussian Splatting in surface reconstruction, specifically the ambiguity of normals and the difficulty of extracting clean meshes from volumetric ellipsoids [S1][S9, unofficial]. By replacing 3D Gaussians with oriented 2D disks, the method leverages the geometric precision of surfels while maintaining the differentiable rendering efficiency of Gaussian Splatting [S4][S9, unofficial]. It has since become a foundational representation for surface-aware Gaussian splatting variants [S1][S6, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] SparSplat: Fast Multi-View Reconstruction with Generalizable 2D Gaussian Splatting, https://doi.org/10.48550/arXiv.2505.02175, fetched 2026-10-26 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S4] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S5] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S7] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S8] GAINS: Gaussian-Based Inverse Rendering from Sparse Multi-view Captures, https://doi.org/10.1007/978-3-032-37602-2_18, fetched 2026-10-29
[S9] Gaussian Splats Part 2: From Photos to a Trained Model, https://alphapixeldev.com/gaussian-splats-part-2-creating-gaussian-splats/, fetched 2026-10-29 (unofficial)
