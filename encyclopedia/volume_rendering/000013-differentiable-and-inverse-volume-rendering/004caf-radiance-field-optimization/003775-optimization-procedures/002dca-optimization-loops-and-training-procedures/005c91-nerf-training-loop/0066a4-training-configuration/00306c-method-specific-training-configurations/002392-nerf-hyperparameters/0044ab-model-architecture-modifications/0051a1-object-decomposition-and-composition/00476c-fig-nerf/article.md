# Fig-NeRF

Also called: FigNeRF

## Overview
Fig-NeRF is a scene composition model designed for object interpolation and amodal segmentation by decomposing a scene into distinct foreground and background components [S1]. It differs from monolithic NeRFs by employing two separate neural radiance field models: a deformable model for the foreground object and a latent-conditioned appearance model for the background, enabling explicit manipulation of object shape and appearance [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images of objects from a specific category; a latent code $z$ representing the object instance.
*   **Outputs:** Rendered images of the foreground object and the background scene, allowing for amodal segmentation and object interpolation [S1].

**Data Structures:**
*   **Foreground Model:** A deformable NeRF architecture (specifically the Nerfies model) [S1]. It maps 3D coordinates and viewing directions to density and color, but includes a deformation field conditioned on the latent code $z$ [S2, unofficial].
*   **Background Model:** An appearance NeRF conditioned on latent codes [S1]. It represents the static or semi-static background environment [S1].
*   **Latent Code $z$:** A vector that conditions both the foreground deformation and the background appearance [S1][S2, unofficial].

**Steps:**
1.  **Latent Encoding:** Encode the input object instance into a latent code $z$ [S2, unofficial].
2.  **Foreground Deformation:** Use the latent code $z$ to condition a template shape of the object category, generating a deformation field that warps the canonical coordinates of the foreground object [S2, unofficial]. This allows the network to explain shape variations as intuitive deformations of a template [S2, unofficial].
3.  **Background Appearance:** Use the latent code $z$ to condition the appearance of the background model [S1].
4.  **Rendering:** Render the foreground and background separately using volume rendering equations [S1].
5.  **Loss Calculation:** Compute two separate photometric losses: one for the foreground rendering and one for the background rendering [S1].
6.  **Optimization:** Optimize the parameters of both NeRF models and the latent code $z$ to minimize the combined photometric losses [S1].

**Formulas:**
*   **Foreground Rendering:** $I_{fg} = \int_{t_n}^{t_f} T(t) \mathbf{c}_{fg}(\mathbf{x}(t), \mathbf{d}) \alpha(t) dt$, where $\mathbf{x}(t)$ is deformed by the latent-conditioned deformation field [S1][S2, unofficial].
*   **Background Rendering:** $I_{bg} = \int_{t_n}^{t_f} T(t) \mathbf{c}_{bg}(\mathbf{x}(t), \mathbf{d}, z) \alpha(t) dt$, where $\mathbf{c}_{bg}$ is conditioned on $z$ [S1].
*   **Total Loss:** $\mathcal{L} = \mathcal{L}_{photometric}(I_{fg}, I_{fg}^{gt}) + \mathcal{L}_{photometric}(I_{bg}, I_{bg}^{gt})$ [S1].

**Parameters:**
*   **Latent Dimension:** Not given in the sources.
*   **Deformation Field Complexity:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Amodal Segmentation:** The method assumes the background model can effectively segment the object from its background, which may fail if the object and background have similar appearances or if the latent code does not sufficiently disentangle them [S1].
*   **Deformation Artifacts:** The deformable foreground model may produce artifacts if the latent code $z$ leads to extreme or unrealistic deformations of the template shape [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Achieves good results for amodal segmentation and object interpolation on datasets such as ShapeNet, Gelato, and Objectron [S1].
*   **Strengths:** Enables explicit control over object shape and appearance through latent codes; supports object interpolation and amodal segmentation [S1][S2, unofficial].
*   **Limitations:** Requires separate models for foreground and background, increasing complexity; relies on the effectiveness of the latent code in disentangling object and background features [S1].

## Background
Fig-NeRF was introduced in April 2021 [S1]. It builds on the NeRFies deformable NeRF model [S1] and focuses on scene composition, specifically object interpolation and amodal segmentation [S1]. It models an object category as a template shape conditioned on a latent code, which undergoes a deformation also conditioned on the same latent variable [S2, unofficial]. This approach allows the network to explain shape variations as intuitive deformations [S2, unofficial]. The method was applied to datasets including ShapeNet, Gelato, and Objectron [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
