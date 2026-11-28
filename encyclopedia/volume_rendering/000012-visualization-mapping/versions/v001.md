# Visualization mapping
## Overview
Visualization mapping constitutes the semantic layer of volume rendering, responsible for translating raw volumetric data values into optical properties and visual styles. It solves the problem of visual ambiguity in 3D datasets by determining which structures are visible, how they appear, and how users interact with the representation [S1, S2, S3, unofficial][S4]. This family is distinct from data representation [2-volume-data-and-representation.md] and rendering algorithms [16-image-order-rendering.md], [22-object-order-rendering.md] by focusing exclusively on the mapping from data space to visual space. It differs from [Surface extraction](29-surface-extraction.md) by operating on continuous volumetric samples rather than discrete isosurfaces, and from [Light transport in participating media](32-light-transport-in-participating-media.md) by prioritizing perceptual clarity and feature emphasis over physical accuracy [S3, unofficial].
## Shared principles
All methods in this family evaluate a mapping function $M(\mathbf{x})$ at sample points $\mathbf{x}$ within the volume to produce visual attributes. The core mechanism involves:
1.  **Input Evaluation**: Sampling the data field $f(\mathbf{x})$ and potentially derived features (e.g., gradients, eigenvalues) at position $\mathbf{x}$.
2.  **Mapping**: Applying a function to convert these inputs into visual parameters. For scalar data, this typically yields opacity $\alpha \in [0, 1]$ and color $\mathbf{c} \in [0, 1]^3$ [S5]. For vector/tensor data, it yields directional cues or geometric primitives [39-multi-field-vector-and-tensor-volume-visualization.md].
3.  **Integration**: These parameters are used during the rendering pass (e.g., in the volume rendering integral) to modulate the contribution of each sample to the final pixel color.
4.  **Feedback**: In interactive contexts, user inputs modify the parameters of $M(\mathbf{x})$ in real-time [41-interaction.md].
## Subtopics

### [Classification and transfer functions](38-classification-and-transfer-functions.md)
Maps scalar data values to opacity and color. It sets itself apart by focusing on the mathematical definition of visibility thresholds (binary vs. soft) and the implementation structures (LUTs, piecewise functions) for scalar fields. Choose this for standard direct volume rendering of scalar datasets.

### [Multi-field, vector and tensor volume visualization](39-multi-field-vector-and-tensor-volume-visualization.md)
Maps non-scalar data (vectors, tensors, complex numbers) to visual attributes. It differs by encoding directional, anisotropic, or phase-dependent information using glyphs or color-based mappings of field structure. Choose this for diffusion tensor imaging (DTI) or fluid flow visualization.

### [Illustrative and non-photorealistic volume rendering](40-illustrative-and-non-photorealistic-volume-rendering.md)
Augments physics-based rendering with artistic techniques to enhance structure perception. It sets itself apart by using local volumetric properties (gradients) to define features rather than hard boundaries, requiring less manual tuning than traditional classification. Choose this when physical accuracy is less important than structural clarity.

### [Interaction](41-interaction.md)
Provides the control loop between the user and the renderer. It differs by focusing on the manipulation of parameters (geometric, data, visual) rather than the static mapping itself. Choose this for applications requiring real-time exploration and parameter adjustment.

## Background
The concept of mapping data to visual properties originated with the development of Direct Volume Rendering (DVR) in the 1980s. Early work focused on simple transfer functions to define opacity and color for scalar data [S1, S2, S3, unofficial][S4]. The field expanded to include multi-field visualization as vector and tensor data became common in scientific computing [39-multi-field-vector-and-tensor-volume-visualization.md]. Non-photorealistic techniques were later introduced to address the "visual clutter" inherent in dense volumetric data [S3, unofficial]. Interaction frameworks evolved to support real-time parameter adjustment, becoming standard in modern visualization systems [41-interaction.md].
## Sources
[S1] Classification and transfer functions overview, URL, date fetched
[S2] Classification and transfer functions overview, URL, date fetched
[S3] Classification and transfer functions overview, URL, date fetched
[S4] Classification and transfer functions overview, URL, date fetched
[S5] Opacity definition, URL, date fetched
[S6] Weight definition, URL, date fetched
[S8] Pre-classification vs Post-classification, URL, date fetched
[S3857] Illustrative rendering framework, URL, date fetched
