# Relightable Gaussian Codec Avatars

Also called: GHG, GoHA, GoHA / GHG, Relightable Gaussian head methods using FLAME rig

## Overview
Relightable Gaussian Codec Avatars is a method for creating photorealistic, animatable 3D head avatars that support dynamic relighting [S2, unofficial]. It builds on [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by integrating a relightable appearance model with the FLAME parametric head rig [S1, unofficial]. It distinguishes itself from standard Gaussian Avatars by explicitly modeling lighting interactions, allowing the avatar to be rendered under diverse, novel lighting conditions rather than being fixed to the capture lighting [S2, unofficial].

## How it works
**Inputs:** Multi-angle video capture of a subject (typically ~30 seconds) [S1, unofficial].
**Outputs:** A compressed 3D Gaussian representation (≤30 MB) with pose-, expression-, and lighting-dependent parameters [S1, unofficial].

**Data Structures:**
1.  **FLAME Model:** Provides the base mesh topology and deformation fields for pose ($\theta$) and expression ($\beta$) [S1, unofficial].
2.  **Gaussian Primitives:** Initialized per mesh triangle in the tangent space of the FLAME mesh [S1, unofficial]. Each Gaussian $i$ stores:
    *   Position $\mathbf{p}_i$ (3D)
    *   Rotation $\mathbf{q}_i$ (Quaternion, 4D)
    *   Scale $\mathbf{s}_i$ (3D)
    *   Opacity $\alpha_i$ (Scalar)
    *   Appearance coefficients (for BRDF/Color)
3.  **Deformation MLP:** A per-Gaussian or global network that predicts pose- and expression-conditioned offsets to the base Gaussian parameters [S1, unofficial].
4.  **Relightable BRDF:** A neural or parametric BRDF model that maps surface normals, view direction, and light direction to color [S1, unofficial].

**Steps:**
1.  **Capture & Fit:** Capture multi-angle video. Run a fitting algorithm (e.g., MICA or DECA) to estimate initial FLAME parameters ($\theta, \beta$) for each frame [S1, unofficial].
2.  **Initialization:** Initialize one 3D Gaussian per triangle of the FLAME mesh. Set initial positions on the mesh surface, rotations aligned with the mesh normal, and scales based on triangle size [S1, unofficial].
3.  **Geometry Optimization:** Train the Gaussian parameters (position, rotation, scale, opacity) using pose-conditioned offsets. The deformation network predicts offsets $\Delta \mathbf{p}, \Delta \mathbf{q}, \Delta \mathbf{s}$ conditioned on $\theta$ and $\beta$ [S1, unofficial].
    *   Optimized position: $\mathbf{p}'_i = \mathbf{p}_i + \Delta \mathbf{p}_i(\theta, \beta)$
    *   Optimized rotation: $\mathbf{q}'_i = \mathbf{q}_i \otimes \Delta \mathbf{q}_i(\theta, \beta)$
    *   Optimized scale: $\mathbf{s}'_i = \mathbf{s}_i \cdot \Delta \mathbf{s}_i(\theta, \beta)$
4.  **Appearance & Relighting:** Train a relightable appearance model. This involves optimizing the BRDF parameters or neural weights such that the rendered color $\mathbf{C}$ depends on the incident light direction $\mathbf{l}$ and view direction $\mathbf{v}$ [S1, unofficial].
    *   $\mathbf{C} = f_{\text{BRDF}}(\mathbf{n}, \mathbf{v}, \mathbf{l}, \text{albedo}, \text{roughness}, \dots)$
5.  **Compression:** Compress the trained model to ≤30 MB using techniques like quantization or pruning [S1, unofficial].

**Parameters:**
*   Capture duration: ~30 seconds [S1, unofficial].
*   Target file size: ≤30 MB [S1, unofficial].
*   Parametric model: FLAME [S1, unofficial].

**Edge Cases:**
*   Sparse views can lead to local minima in optimization [S2, unofficial].
*   Precision issues in quaternion normalization during deformation [G].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Training time depends on the number of Gaussians and the complexity of the deformation/BRDF networks. Compression is fast.
*   **Memory Complexity:** High during training; final model size is compressed to ≤30 MB [S1, unofficial].
*   **Performance:** Supports real-time rendering if optimized for rasterization. Relighting adds computational cost compared to fixed-lighting methods [S2, unofficial].
*   **Strengths:** High photorealism, support for novel lighting, compact representation [S1, S2, unofficial].
*   **Limitations:** Requires multi-angle capture; fitting FLAME can be sensitive to initialization [S1, unofficial].

## Background
Relightable Gaussian Codec Avatars was introduced to support more diverse lighting conditions in Gaussian avatar generation [S2, unofficial]. It is part of the evolution of Gaussian Avatar Reconstruction methods, following approaches like GaussianAvatars and GoMAvatar, which focused on geometry and deformation but not explicit relighting [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
