# Fig-NeRF

Also called: FigNeRF

## Overview
Fig-NeRF is a scene composition model designed for object interpolation and amodal segmentation by partitioning the scene into distinct foreground and background components [S1]. It differs from monolithic NeRFs by using two separate models: a deformable NeRF for the foreground object and a latent-conditioned appearance NeRF for the background, enabling explicit manipulation of object shape and appearance [S1].

## How it works
**Inputs:** A set of multi-view images of an object category, potentially with associated 2D masks or semantic labels for supervision [S1].
**Outputs:** Two radiance fields: a foreground field representing the deformable object and a background field representing the scene context [S1].

**Data Structures:**
1.  **Foreground Model:** A deformable NeRF architecture (specifically the Nerfies model) [S1]. It maps 3D coordinates and viewing directions to density and color, incorporating a deformation field [S1].
2.  **Background Model:** An appearance NeRF conditioned on a latent code [S1]. It maps 3D coordinates and viewing directions to density and color, with appearance variations controlled by the latent vector [S1].
3.  **Latent Code:** A learnable vector $\mathbf{z}$ that conditions both the object's template shape and its deformation [S2, unofficial].

**Steps:**
1.  **Latent Encoding:** For a given object instance, encode its shape and appearance variations into a latent code $\mathbf{z}$ [S2, unofficial].
2.  **Foreground Deformation:** Use $\mathbf{z}$ to condition a template shape of the object category. Apply a deformation field, also conditioned on $\mathbf{z}$, to map canonical 3D coordinates to the observed object geometry [S2, unofficial]. This allows the network to explain shape variations as intuitive deformations [S2, unofficial].
3.  **Background Conditioning:** Condition the background appearance NeRF on the same latent code $\mathbf{z}$ (or a separate background-specific latent) to capture scene context and appearance variations [S1].
4.  **Rendering:** Render the foreground and background components separately using volume rendering [S1].
5.  **Supervision:** Compute two separate photometric losses: one for the foreground object and one for the background [S1]. Use a learnable background model to assist in segmenting the object from the background [S2, unofficial].

**Formulas:**
*   Foreground Radiance: $\mathbf{C}_f(\mathbf{x}, \mathbf{d}) = \text{DeformableNeRF}(\mathbf{x}, \mathbf{d}, \mathbf{z})$
*   Background Radiance: $\mathbf{C}_b(\mathbf{x}, \mathbf{d}) = \text{AppearanceNeRF}(\mathbf{x}, \mathbf{d}, \mathbf{z})$
*   Total Loss: $\mathcal{L} = \mathcal{L}_{photometric}^{foreground} + \mathcal{L}_{photometric}^{background}$ [S1]

**Parameters:**
*   Latent dimension: Not given in the sources.
*   Deformation field complexity: Based on Nerfies [S1].

**Edge Cases:**
*   Amodal segmentation requires handling occluded parts of the object, which the deformable model must infer from visible parts [S1].
*   Separation between foreground and background relies on the distinct conditioning and loss functions; ambiguity in boundary regions may require additional mask supervision [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Training involves optimizing two NeRFs and a latent code, likely slower than a single NeRF due to the dual-model structure and deformation field [S1].
*   **Memory Complexity:** Higher than standard NeRF due to storing parameters for both foreground and background models [S1].
*   **Strengths:** Enables object interpolation and amodal segmentation; disentangles object shape/appearance from background [S1].
*   **Limitations:** Requires careful balancing of two photometric losses; deformation field may be sensitive to initialization [S1][S2, unofficial].
*   **When to use:** When object-level manipulation, interpolation, or segmentation is required [S1].
*   **When not to use:** For simple novel view synthesis without object-level control, where a single NeRF is more efficient [S1].

## Background
Fig-NeRF was introduced in April 2021 [S1]. It focuses on scene composition for object interpolation and amodal segmentation [S1]. It builds on the Nerfies deformable NeRF model for the foreground [S1]. It was evaluated on datasets such as ShapeNet, Gelato, and Objectron [S1]. The approach models an object category as a template shape conditioned on a latent code, with deformation also conditioned on the same latent variable [S2, unofficial].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
