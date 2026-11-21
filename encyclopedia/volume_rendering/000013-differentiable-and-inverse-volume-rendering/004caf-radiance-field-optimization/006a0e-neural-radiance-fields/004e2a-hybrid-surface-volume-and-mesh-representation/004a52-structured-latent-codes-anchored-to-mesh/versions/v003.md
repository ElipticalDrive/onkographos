# Structured Latent Codes Anchored to Mesh

Also called: anchor mesh latent codes, l_geo / l_rgb

## Overview
This method stores structured geometry ($l_{geo}$) and appearance ($l_{rgb}$) latent codes at anchor nodes (typically mesh vertices) on a deformable mesh, enabling local editing and deformation tracking [S1, unofficial]. It solves the ambiguity of point-to-latent mapping in 3D space by anchoring codes to a mesh surface, allowing the density field to track vertex movements for geometry editing [S1, unofficial]. Unlike sibling methods that may use global parametric controls or unstructured point clouds, this approach leverages the mesh structure to support fine-grained, local manipulations such as bulges or texture transfers while maintaining 3D consistency [S1, unofficial].

## How it works
**Inputs:** A canonical mesh (e.g., SMPL in rest pose) defining anchor nodes (vertices); a 3D query point $x_t$ in target pose; Top-K nearest anchor vertices $\{v_i\}_{i=1}^K$; and pre-trained latent codes $l_{geo}^i, l_{rgb}^i$ stored at each anchor node.
**Outputs:** Interpolated latent codes $l_{geo}, l_{rgb}$ for the query point; deformed position $LBS(x_t)$.

**Data Structures:**
*   **Anchor Mesh:** A set of vertices $V$. Each vertex $v_i$ has an associated latent code vector.
*   **Latent Codes:** Vectors $l_{geo}^i, l_{rgb}^i \in \mathbb{R}^{32}$ stored at each vertex $v_i$ [S1, unofficial].

**Steps:**
1.  **Identify Nearest Anchors:** For a query point $x_t$ in the target pose, determine the $K$ nearest vertices $\{v_i\}_{i=1}^K$ on the canonical mesh [S1, unofficial].
2.  **Compute Weights:** Calculate inverse distance weights $w_i$ for each neighbor:
    $$ w_i = \frac{1}{\|x_t - v_i\|^2} $$
    Normalize weights to sum to 1:
    $$ \tilde{w}_i = \frac{w_i}{\sum_{j=1}^K w_j} $$
    (Note: The source defines $w_i$ as the unnormalized inverse square distance in Eq. 1, then uses normalized weights for interpolation [S1, unofficial]).
3.  **Interpolate Latent Codes:** Compute the interpolated geometry and appearance codes using the normalized weights:
    $$ l_{geo} = \sum_{i=1}^K \tilde{w}_i l_{geo}^i $$
    $$ l_{rgb} = \sum_{i=1}^K \tilde{w}_i l_{rgb}^i $$
    [S1, unofficial].
4.  **Deform Position (Optional for Rendering):** To obtain the deformed position for rendering, interpolate the Linear Blend Skinning (LBS) weights $b^i$ from the anchor nodes using the same weights $\tilde{w}_i$, then apply the skinning transformation:
    $$ LBS(x_t) = \left( \sum_{i=1}^K \tilde{w}_i B^i \right)^{-1} x_t $$
    where $B^i$ is the skinning matrix for vertex $i$ [S1, unofficial].
5.  **Resolve Ambiguity (Signed Height):** To handle points perpendicular to the surface, introduce a local signed height indicator $\tilde{h}$ based on the signed distance and UV coordinates of $x_t$, mapping the 3D point to a 2.5D volume [S1, unofficial].

**Parameters:**
*   **Latent Dimension:** 32 for both $l_{geo}$ and $l_{rgb}$ [S1, unofficial].
*   **K:** Number of nearest neighbors (Not given in the sources).
*   **Learning Rate:** $5 \times 10^{-4}$ decaying to $5 \times 10^{-6}$ for networks; $10\times$ higher for latent codes [S1, unofficial].

**Edge Cases:**
*   **Perpendicular Ambiguity:** Points along the normal direction may have identical latent codes but different positions. Mitigated by the signed height indicator $\tilde{h}$ [S1, unofficial].
*   **Boundary Effects:** For texture painting, masks are dilated to ensure smooth boundaries during fine-tuning [S1, unofficial].

## Subtopics

### [Texture Swapping](12720-texture-swapping.md)
Texture swapping transfers appearance latent codes from a source mesh area to a target mesh area within a trained Unified Volumetric Avatar (UVA) or similar neural mesh representation. It solves the problem of reusing textures between objects with slightly different geometries by performing non-rigid 3D alignment of anchor nodes (mesh vertices). Unlike Texture Painting, which modifies appearance directly on a single model, swapping operates between two distinct trained models or areas, leveraging pre-blended latent codes for better 3D consistency.

### [Texture Painting](12730-texture-painting.md)
Texture painting is a method for locally editing the appearance of a 3D avatar or mesh by fine-tuning specific latent codes to match a 2D input painting. It solves the problem of transferring 2D artistic intent to a 3D neural representation while preserving the geometry and appearance of unedited regions. Unlike global texture replacement, it operates on a sparse set of anchor nodes, allowing for independent, spatial-aware local edits.

### Mesh-Guided Geometry Editing
**Definition:** An editing technique allowing local manipulation of avatar geometry by moving anchor nodes on the mesh, with the density field tracking these movements [S1, unofficial].
**Steps:**
1.  Users move anchor nodes in 3D modeling software (e.g., Blender) or using mesh deforming methods (e.g., ARAP) [S1, unofficial].
2.  Copy movements of corresponding vertices from the canonical mesh to the target mesh to deform the skinning field in sync for animation [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:** Memory usage scales with the number of mesh vertices (anchor nodes) and the latent dimension (32). Interpolation is $O(K)$ per query point, where $K$ is the number of neighbors. Training requires joint optimization of motion and canonical fields (e.g., 300K iterations) [S1, unofficial].
**Measured Performance:** Training takes about 10 hours using two NVIDIA Tesla V100 GPUs for 9 performers from the ZJU-MoCap dataset with a batch size of 1024 [S1, unofficial].
**Strengths:** Enables local geometry editing (e.g., bulges) and texture painting while maintaining 3D consistency and novel view synthesis [S1, unofficial]. The mesh-based skinning field allows synchronized deformation for animation [S1, unofficial].
**Limitations:** Interpolation alone can lead to ambiguity for points perpendicular to the surface, requiring additional indicators like signed height [S1, unofficial]. Fine-tuning the texture decoder locally is non-trivial [S1, unofficial].

## Background
Originated in the Unified Volumetric Avatar (UVA) method [S1, unofficial]. The method builds on the idea that a skinning field is naturally a mesh-based representation, making it possible to track vertex movements and desired geometry editing [S1, unofficial]. It uses parametric models like SMPL as the anchored mesh baseline [S1, unofficial].

## Sources
[S1] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
