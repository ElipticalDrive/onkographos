# Structured Latent Codes Anchored to Mesh

Also called: anchor mesh latent codes, l_geo / l_rgb

## Overview
This method represents scene geometry and appearance using structured latent codes ($l_{geo}$ and $l_{rgb}$) stored at anchor nodes on a deformable mesh, enabling local editing and deformation tracking [S1, unofficial]. It sets itself apart from implicit surface methods by explicitly tying latent features to mesh vertices, allowing the density field to track vertex movements for geometry editing, and from surfel methods by using continuous interpolation of latent codes rather than discrete primitive intersections [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A canonical mesh (e.g., SMPL rest pose) with vertices serving as anchor nodes; a 3D point $\mathbf{x}$ in canonical space; target pose parameters $\theta_t$.
*   **Output:** Geometry latent code $l_{geo}$ and appearance latent code $l_{rgb}$ for point $\mathbf{x}$; deformed point position.

**Data Structures:**
*   **Anchor Nodes:** Vertices of the canonical mesh.
*   **Latent Codes:** Vectors of dimension 32 for both $l_{geo}$ and $l_{rgb}$ [S1, unofficial].
*   **Signed Height Indicator:** A local signed height $\tilde{h}$ combined with UV coordinates to resolve ambiguity for points perpendicular to the surface [S1, unofficial].

**Steps:**
1.  **Identify Nearest Neighbors:** For a 3D point $\mathbf{x}_t$ in the target pose, determine the $K$ nearest mesh vertices $\{v_i\}_{i=1}^K$ from the canonical mesh [S1, unofficial].
2.  **Compute Weights:** Calculate inverse distance weights $w_i$ for each neighbor:
    $$ w_i = \frac{1}{\|\mathbf{x}_t - v_i\|^2} $$
    Normalize weights:
    $$ \bar{w}_i = \frac{w_i}{\sum_{j=1}^K w_j} $$
    [S1, unofficial]
3.  **Interpolate Latent Codes:** Compute the latent code $l_{geo/rgb}$ for point $\mathbf{x}_t$ by interpolating the latent codes of the $K$ nearest anchor nodes using the normalized weights:
    $$ l_{geo/rgb} = \sum_{i=1}^K \bar{w}_i l_{geo/rgb}^i $$
    [S1, unofficial]
4.  **Resolve Ambiguity:** Use the local signed height indicator $\tilde{h}$ (derived from signed distance and UV coordinates) to disambiguate points along the surface normal, mapping 3D points to a 2.5D volume representation [S1, unofficial].
5.  **Deformation (Optional for Rendering):** If rendering in a target pose, deform the point using the interpolated Local Blend Skinning (LBS) weights $\mathbf{b}$:
    $$ \text{LBS}(\mathbf{x}_t) = \mathbf{W}(\mathbf{b})^{-1} \mathbf{x}_t $$
    where $\mathbf{W}(\mathbf{b})$ is the transformation matrix constructed from the interpolated skinning weights [S1, unofficial].

**Parameters:**
*   **Latent Code Dimension:** 32 [S1, unofficial].
*   **K:** Number of nearest neighbors (Not given in the sources).
*   **Network Architectures:** Appearance network: 8-layer MLP with skip connection at 4th layer. Density network: 4-layer MLP with softplus activation. Shading network: 3-layer MLP [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Perpendicular Ambiguity:** Points along the direction perpendicular to the surface may have the same latent codes but different positions. This is addressed by introducing the signed height indicator [S1, unofficial].
*   **Mesh Selection:** The anchored mesh can be any form (parametric or off-the-shelf), but SMPL is used as the baseline [S1, unofficial].

## Subtopics

### [Texture Swapping](12720-texture-swapping.md)
Swapping textures between two trained UVAs by selecting anchor nodes of interest and performing non-rigid 3D alignment [12720-texture-swapping.md]. Choose this when transferring appearance attributes between different avatar instances while maintaining pose consistency.

### [Texture Painting](12730-texture-painting.md)
Editing texture by fine-tuning locally selected latent codes to match a 2D painting, using a binary mask and point-to-ray distance [12730-texture-painting.md]. Choose this for direct artistic control over specific regions of the avatar's surface.

### Mesh-Guided Geometry Editing
Editing technique allowing local manipulation of avatar geometry by moving anchor nodes on the mesh, with the density field tracking these movements [S1, unofficial]. Users move anchor nodes in 3D modeling software (e.g., Blender) or using mesh deforming methods (e.g., ARAP) [S1, unofficial]. The movements of corresponding vertices are copied from the canonical mesh to the target mesh to deform the skinning field in sync for animation [S1, unofficial]. Choose this when fine-grained, local geometric modifications are required beyond global shape parameters.

## Operating characteristics
*   **Time Complexity:** Depends on the number of nearest neighbors $K$ and the depth of the MLPs. Training takes about 10 hours using two NVIDIA Tesla V100 GPUs for 300K iterations with a batch size of 1024 [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of mesh vertices (anchor nodes) times the latent code dimension (32) [S1, unofficial].
*   **Strengths:** Enables local geometry and texture editing; tracks deformation naturally via mesh-based representation; reduces ambiguity with signed height indicators [S1, unofficial].
*   **Limitations:** Requires a predefined mesh structure (e.g., SMPL); interpolation may introduce smoothing artifacts if $K$ is not chosen carefully [S1, unofficial].
*   **When to Use:** When local editing of geometry or texture is required, or when animation fidelity needs to be preserved through mesh-based deformation [S1, unofficial].
*   **When Not to Use:** For scenes without a clear parametric body model or when global shape changes are the primary requirement [S1, unofficial].

## Background
Originated in the UVA (Unified Volumetric Avatar) paper, which proposes a method for unified volumetric avatar representation [S1, unofficial]. The method leverages the skinning field as a mesh-based representation to track vertex movements, enabling geometry editing and animation [S1, unofficial]. It builds on the principles of hybrid surface-volume representations by anchoring latent codes to explicit mesh structures [S1, unofficial].

## Sources
[S1] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
