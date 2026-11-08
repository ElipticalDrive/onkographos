# Neural Textures Architecture

Also called: Neural Texture Models, Per-Mesh Neural Textures

## Overview
Neural Textures Architecture is a method for encoding view-dependent appearance onto 3D mesh surfaces using small, per-mesh neural networks. It solves the problem of representing high-frequency, view-dependent lighting effects (such as specularities and subsurface scattering) that cannot be captured by static albedo textures. It differs from its parent, the Color Network Architecture, by explicitly decoupling the representation into separate models for RGB and transparency per mesh, and by outputting spherical harmonics (SH) coefficients rather than direct color values, allowing for efficient view-dependent decoding via the view direction vector.

## How it works
The architecture operates during the mesh texturing phase. It maps 2D surface coordinates to view-dependent appearance properties.

**Inputs and Outputs:**
*   **Input:** 2D UV coordinates $(u, v)$ representing the position on the mesh surface.
*   **Output:** Per-channel spherical harmonics (SH) coefficients of degree 3. These coefficients are subsequently decoded using the view direction $v$ to produce RGB color and transparency.

**Data Structures and Layout:**
The neural model is a Multi-Layer Perceptron (MLP) with the following specific layout:
1.  **Positional Encoding:** The input UV coordinates are encoded using a trainable positional encoding scheme [S1]. This maps the 2D coordinates into a higher-dimensional space to capture high-frequency details.
2.  **MLP Structure:** The network consists of two hidden layers.
    *   Each hidden layer contains 64 features (neurons).
    *   Hidden layers use ReLU activations.
    *   The final layer uses a linear activation function.
3.  **Output Dimension:** The final layer outputs a vector of SH coefficients. For degree 3 SH, there are $2 \times 3 + 1 = 7$ coefficients per channel. Assuming RGB (3 channels), the output vector size is $3 \times 7 = 21$ values.

**Steps:**
1.  **Encode UV:** Take the 2D UV coordinate $(u, v)$ for a point on the mesh.
2.  **Apply Positional Encoding:** Pass $(u, v)$ through the trainable positional encoding function $\gamma(u, v)$ [S1].
3.  **Forward Pass:** Pass the encoded vector through the two-layer MLP:
    $$ h_1 = \text{ReLU}(W_1 \gamma(u, v) + b_1) $$
    $$ c = W_2 h_1 + b_2 $$
    Where $W_1, W_2$ are weight matrices and $b_1, b_2$ are bias vectors. The output $c$ is the vector of SH coefficients.
4.  **Decode View Dependence:** The SH coefficients $c$ are combined with the view direction vector $v$ to compute the final color and transparency. The specific decoding formula for SH is not detailed in the source, but it involves projecting the view direction onto the SH basis functions defined by the coefficients [S1].

**Parameters:**
*   **Hidden Units:** 64 per layer [S1].
*   **SH Degree:** 3 [S1].
*   **Activation:** ReLU for hidden layers, Linear for output [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The source notes that gradients of SDFs (used in the broader context of the method) are computed with finite differences using $\epsilon = 10^{-4}$ [S1]. This precision setting is critical for the stability of the geometry optimization that precedes texturing.
*   **Boundary Conditions:** The method assumes valid UV coordinates. Degenerate UV mappings (e.g., severe stretching or overlapping) may lead to artifacts, though specific mitigation strategies are not detailed in the provided text [S1].

## Subtopics

### SIREN Optimization for Neural Lumigraph
This variant uses SIREN (Sinusoidal Representation Networks) architectures for the SDF $S$ and emissivity $E$ to enable smooth, high-frequency interpolation. It is chosen when favorable angular interpolation behavior is required for neural lumigraph rendering. It differs from the standard ReLU-based Neural Textures by using Fourier features and sinusoidal activations to support angular resolution.
*   **Implementation:** MLPs for $S$ and $E$ have 5 layers with 256 hidden units each [S2].
*   **Features:** Fourier features $\{\sin(2k\pi r_d), \cos(2k\pi r_d) \mid k \in 1 \dots 4\}$ are used in $E$ to support angular resolution [S2].
*   **Initialization:** $S$ is initialized to a unit sphere of radius 0.5 by pretraining to a procedural shape [S2].
*   **Optimization:** Loss is optimized in mini-batches of 50,000 rays sampled uniformly. Adam solver with initial learning rate $10^{-4}$, decreased by a factor of 2 every 40,000 batches for 150,000 batches total [S2].
*   **Loss Function:** The total loss is $\arg \min_{\theta, \phi} L_R + w_E L_E + w_M L_M + w_S L_S$ [S2].
*   **Angular Smoothness:** The angular smoothness loss is $L_S = \frac{1}{|U|} \sum ||\nabla^2_{r_d} E(x, r_d, n; \theta, \phi)||^2_2$ [S2].
*   **Parameters:** $w_E = 0.1$, $w_M = 100$, $w_S = 0.01$ [S2].
*   **Pitfalls:** Local overfitting of SIREN, especially for high-frequency function $E$, is prevented by using large batch sizes and uniform ray distribution [S2]. Large values of $w_S$ cause high-frequency artifacts in $S$ [S2].

## Operating characteristics
*   **Complexity:** The MLP is small (2 layers, 64 units), making inference computationally cheap per pixel. However, the positional encoding is trainable, requiring gradient updates during training.
*   **Memory:** Memory usage is proportional to the number of meshes and the size of the SH coefficient vectors. For degree 3 SH, this is 21 floats per point per mesh.
*   **Performance:** The method allows for compact representation of view-dependent effects. The use of SH coefficients allows for efficient view-dependent rendering without storing full view-dependent textures.
*   **Strengths:** Compact representation, view-dependent, compatible with standard mesh rendering pipelines.
*   **Limitations:** Limited to mesh-based representations. Requires UV unwrapping. The source does not provide specific frame rates or memory benchmarks for this specific architecture [S1].

## Background
The Neural Textures Architecture was introduced in the context of "Volumetric Surfaces: Representing Fuzzy Geometries with Layered Meshes" [S1]. It builds upon the concept of neural textures [35] (Muller et al.) and adapts it for volumetric rendering of fuzzy geometries. The method separates the representation of RGB and transparency per mesh to handle the complexities of volumetric data. The SIREN variant was introduced in "Neural Lumigraph Rendering" [S2] to improve angular interpolation for novel view synthesis.

## Sources
[S1] Volumetric Surfaces: Representing Fuzzy Geometries with Layered Meshes, https://doi.org/10.1109/cvpr52734.2025.01991, fetched 2026-10-26
[S2] Neural Lumigraph Rendering, https://doi.org/10.1109/cvpr46437.2021.00427, fetched 2026-10-26
