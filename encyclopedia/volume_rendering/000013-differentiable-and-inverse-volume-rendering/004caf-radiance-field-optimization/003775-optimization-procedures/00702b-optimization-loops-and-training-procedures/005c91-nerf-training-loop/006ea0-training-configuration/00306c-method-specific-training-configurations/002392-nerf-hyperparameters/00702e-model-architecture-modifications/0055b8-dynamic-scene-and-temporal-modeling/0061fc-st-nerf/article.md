# ST-NeRF

Also called: Spatially and temporally consistent NeRF

## Overview
ST-NeRF is a layered radiance field representation designed for editable free-viewpoint videos of dynamic scenes containing independently moving or articulated objects [S1, unofficial][S2]. It solves the problem of spatial and temporal consistency in dynamic neural rendering by disentangling the position, deformation, and appearance of each entity into separate continuous functions spanning space and time [S1, unofficial][S2]. Unlike monolithic dynamic NeRFs that model the entire scene as a single time-dependent field, ST-NeRF uses a layered architecture where each object is a distinct entity, enabling specific editing operations such as rescaling, shifting, duplication, removal, and re-timing [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of synchronized multi-view videos (typically 16 cameras arranged in a half-circle) and human-background segmentation masks [S1, unofficial]. The model also accepts timestamps (or frame numbers) to account for temporal evolution [S1, unofficial][S2].
*   **Outputs:** Accumulated density $\sigma$ and view-dependent color $\mathbf{c}$ for novel views, rendered by integrating through multiple scene layers [S1, unofficial].

**Data Structures:**
The scene is represented as a collection of layers, where each layer corresponds to a dynamic entity (e.g., a performer) [S1, unofficial][S2]. Each entity is defined by a separate continuous function (MLP) that spans space and time [S2]. The architecture consists of two key modules within the MLP:
1.  **Space-Time Deformation Module:** Maps spatial coordinates and time to a canonical space, enforcing space-time coherence [S1, unofficial][S2].
2.  **Neural Radiance Module:** Maps the canonical coordinates and viewing direction to density and color [S1, unofficial][S2].

**Algorithm:**
1.  **Initialization:** For each dynamic entity $i$, initialize a separate MLP $F_i$ composed of the deformation module $D_i$ and the radiance module $R_i$ [S2].
2.  **Encoding Time:** Encode the input timestamp $t$ (or frame number) directly into the model inputs to facilitate disentanglement [S2].
3.  **Deformation:** For a query point $\mathbf{x}$ at time $t$, compute the canonical coordinate $\mathbf{x}'$ using the deformation module:
    $$ \mathbf{x}' = D_i(\mathbf{x}, t) $$
    This step maps the dynamic spatial position to a canonical reference frame, ensuring spatial consistency [S1, unofficial][S2].
4.  **Radiance Field Evaluation:** Compute density $\sigma$ and color $\mathbf{c}$ from the canonical coordinate $\mathbf{x}'$ and viewing direction $\mathbf{d}$ using the radiance module:
    $$ (\sigma, \mathbf{c}) = R_i(\mathbf{x}', \mathbf{d}) $$
5.  **Layered Rendering:** To render a novel view, cast rays through the scene. For each ray, determine intersections with the bounding boxes of the entities [S1, unofficial][S2]. Sample points along the ray within these bounding boxes.
6.  **Accumulation:** Integrate the density and color contributions from all intersected layers along the ray using standard volume rendering equations to produce the final pixel color [S1, unofficial].

**Parameters:**
*   **Camera Setup:** 16 synchronized cameras at regular intervals in a half-circle [S1, unofficial].
*   **Segmentation:** Requires human-background segmentation masks as input [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bounding Box Accuracy:** The efficiency and correctness of the layered rendering depend on accurate bounding boxes for all independently moving objects [S1, unofficial].
*   **Non-Rigid Editing:** The current implementation focuses on rigid transformations (rescaling, shifting, removal) and does not natively support non-rigid scene editing [S1, unofficial].
*   **View Count:** The method relies on a relatively high number of input views (16); reducing this number is noted as a future challenge [S1, unofficial].

## Subtopics

### Layered Radiance Fields
A representation where the scene is decomposed into independent entities, each with its own radiance field, allowing for efficient ray-box intersection and entity-level manipulation [S1, unofficial][S2].

### Space-Time Deformation
A module that maps dynamic coordinates to a canonical space, ensuring that the appearance and geometry remain consistent across time and space [S1, unofficial][S2].

## Operating characteristics
*   **Strengths:** Enables precise neural scene editing including spatial operations (affine transformation, insertion, removal) and temporal operations (re-timing) [S2]. Disentangles position, deformation, and appearance [S1, unofficial].
*   **Limitations:** Requires segmentation masks and multiple synchronized views [S1, unofficial]. Limited to rigid editing of entities defined by bounding boxes; non-rigid editing is not supported [S1, unofficial].
*   **Performance:** Rendering efficiency is improved by focusing only on relevant points where rays intersect bounding boxes [S1, unofficial].
*   **When to use:** When the goal is to edit dynamic scenes (e.g., rescaling, removing, or re-timing specific performers) in free-viewpoint video applications [S1, unofficial][S2].
*   **When not to use:** For scenes with non-rigid deformations or when input views are limited [S1, unofficial].

## Background
ST-NeRF was introduced by Zhang et al. [JXX∗ 21] as a layered representation for editable free-viewpoint videos [S1, unofficial]. It builds on the principles of neural scene graphs and layered radiance fields, where objects are treated as independent entities [S1, unofficial][S2]. The method is part of the broader effort to enable neural scene editing, such as background-foreground decomposition and enriching training datasets for automotive perception [S1, unofficial].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
