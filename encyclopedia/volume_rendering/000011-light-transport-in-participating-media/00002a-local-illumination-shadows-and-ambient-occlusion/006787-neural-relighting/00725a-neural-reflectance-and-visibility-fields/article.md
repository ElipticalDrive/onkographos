# Neural Reflectance and Visibility Fields

Also called: Neural Visibility Fields

## Overview
Neural Reflectance and Visibility Fields (NRF) is a method for novel view synthesis and relighting that decouples scene geometry, material reflectance, and light transport into distinct neural representations [S1, unofficial]. It solves the problem of recovering a relighting model from images with pronounced shadows and self-occlusion by training a Multi-Layer Perceptron (MLP) to approximate light source visibility for any 3D position and incident light direction [S1, unofficial]. Unlike standard NeRF which renders radiance directly, NRF separates the visibility component, enabling efficient recovery of scene appearance under varying lighting conditions [S1, unofficial].

## How it works
The method relies on an MLP that maps spatial coordinates and light directions to visibility values [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A 3D position $\mathbf{x} \in \mathbb{R}^3$ and a 2D incident light direction $\mathbf{d} \in \mathbb{R}^2$ (typically encoded via spherical harmonics or similar parameterization).
*   **Output:** A scalar visibility value $V \in [0, 1]$ representing the fraction of the hemisphere visible from $\mathbf{x}$ in direction $\mathbf{d}$ [S1, unofficial].

**Data Structures:**
*   **Visibility MLP:** A standard feed-forward neural network. The exact architecture (layer sizes, activation functions) is not specified in the provided sources [S1, unofficial].
*   **Query Mechanism:** The network is queried once per incident light direction to determine visibility [S1, unofficial].

**Algorithm Steps:**
1.  **Training Data Preparation:** Collect multi-view images of the scene, potentially with varying lighting or static lighting with strong shadows [S1, unofficial].
2.  **Visibility Network Training:** Train the MLP to predict visibility $V(\mathbf{x}, \mathbf{d})$ by minimizing a loss function that compares rendered images (computed via volume rendering or ray marching using the predicted visibility) against ground-truth images [S1, unofficial]. The specific loss function and rendering equation are not detailed in the sources [S1, unofficial].
3.  **Relighting Query:** For a new lighting condition, query the trained MLP for the visibility of each point in the scene for the new light directions [S1, unofficial].
4.  **Rendering:** Combine the predicted visibility with material properties (if modeled separately) and the new lighting to render the final image [S1, unofficial].

**Formulas:**
*   Visibility Query: $V = \text{MLP}(\mathbf{x}, \mathbf{d})$ [S1, unofficial].
*   The sources do not provide the full volume rendering integral or the specific relationship between visibility, albedo, and final radiance [S1, unofficial].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   The method assumes the visibility field can be adequately represented by an MLP [S1, unofficial].
*   Precision issues with boundary conditions (e.g., points on the surface vs. inside the volume) are not discussed [S1, unofficial].

## Subtopics

### NeuTex Reflectance Fields Extension
This variant extends the neural texture mapping framework to regress reflectance parameters instead of direct radiance [S2]. It modifies the geometry network $F_\sigma$ to jointly regress volume density and surface normal $\mathbf{n}$ at each shading point [S2]. The texture regression network $F_{tex}$ is changed to regress reflectance parameters $\mathbf{r}$ in texture space rather than outputting radiance $\mathbf{c}$ directly [S2]. The central texture mapping and inverse mapping networks remain unchanged [S2]. This approach integrates with reflectance-aware volume rendering, which computes radiance from shading properties (normal, reflectance) under given viewing and lighting conditions [S2]. This allows for high-quality BRDF texture extraction and realistic rendering with BRDF editing [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method enables recovery of a scene's relighting model from images with pronounced shadows and self-occlusion [S1, unofficial]. It requires only one query per incident light direction [S1, unofficial].
*   **Strengths:** Efficient relighting via single query per light direction; handles self-occlusion [S1, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** When relighting from a few viewpoints is required and shadows/occlusions are prominent [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
The concept of Neural Reflectance and Visibility Fields is discussed in the context of neural rendering and its hardware acceleration [S1, S1, unofficial]. It builds upon earlier work in implicit scene representations like DeepSDF and SRNs [S1, unofficial]. The specific variant involving reflectance fields is related to Neural Reflectance Fields [5, 4] which reconstructs BRDFs from flash images [S2]. The NeuTex extension demonstrates the application of neural texture mapping to volumetric neural rendering for BRDF extraction [S2].

## Sources
[S1] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
[S2] NeuTex: Neural Texture Mapping for Volumetric Neural Rendering, https://doi.org/10.1109/cvpr46437.2021.00704, fetched 2026-10-26
