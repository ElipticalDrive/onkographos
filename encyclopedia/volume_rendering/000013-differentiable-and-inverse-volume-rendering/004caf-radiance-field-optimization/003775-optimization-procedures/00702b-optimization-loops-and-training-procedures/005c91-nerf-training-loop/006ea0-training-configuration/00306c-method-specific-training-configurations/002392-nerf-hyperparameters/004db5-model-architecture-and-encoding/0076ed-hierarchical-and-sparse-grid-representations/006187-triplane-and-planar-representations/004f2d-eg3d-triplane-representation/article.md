# EG3D Triplane Representation

Also called: ConvONet / EG3D (Tri-plane), EG3D Triplane-Based 3D GAN, Hybrid Triplane Representation, Tri-plane Features, Tri-plane representation

## Overview
The EG3D Triplane Representation is a 3D scene encoding method that stores learnable features in three orthogonal 2D planes ($xy$, $xz$, $yz$) to enable efficient neural volumetric rendering [S2]. It distinguishes itself from standard 3D grids by reducing memory footprint through 2D convolution operations while maintaining high-resolution detail, and it is specifically optimized for integration into Generative Adversarial Networks (GANs) for 3D-aware image synthesis [S1, unofficial][S2].

## How it works
**Inputs and Outputs**
*   **Input:** A noise vector $z$ and camera pose parameters.
*   **Output:** A triplane tensor $\mathbf{T}$, which is queried to produce feature vectors, color, and density for volume rendering.

**Data Structures**
The core data structure is the triplane tensor $\mathbf{T}$ [S2]:
*   **Shape:** $\mathbb{R}^{256 \times 256 \times 96}$ [S2].
*   **Layout:** This tensor comprises 3 axis-aligned planes ($xy$, $xz$, $yz$) [S2].
*   **Channels:** Each plane has 32 channels ($C=32$) [S2].
*   **Resolution:** Each plane has a spatial resolution of $256 \times 256$ [S2].

**Querying and Rendering**
To render an image, the system performs the following steps [S2]:

1.  **Feature Gathering:** For a query point $\mathbf{x} \in \mathbb{R}^3$, project $\mathbf{x}$ onto each of the three canonical planes ($xy$, $xz$, $yz$).
2.  **Interpolation:** Bilinearly interpolate the features at the projected coordinates on each plane to obtain feature vectors $\mathbf{f}_{xy}$, $\mathbf{f}_{xz}$, and $\mathbf{f}_{yz}$ [S2].
3.  **Aggregation:** Combine the three feature vectors using the mean operator $\Phi$ [S2]:
    $$
    \mathbf{f}_{agg} = \Phi(\mathbf{f}_{xy}, \mathbf{f}_{xz}, \mathbf{f}_{yz}) = \frac{\mathbf{f}_{xy} + \mathbf{f}_{xz} + \mathbf{f}_{yz}}{3}
    $$
4.  **Decoding:** Pass the aggregated feature vector through a lightweight Multi-Layer Perceptron (MLP) to predict the feature vector $\mathbf{f}$, color $c$, and volume density $\sigma$ [S2]:
    $$
    (\mathbf{f}, c, \sigma) = \text{MLP}(\mathbf{f}_{agg})
    $$
    Note that the output values, including color, are independent of viewing direction and depend only on $\mathbf{x}$ [S2].
5.  **Volume Rendering:** Accumulate points along rays using the predicted $\sigma$ and $c$ to perform volume rendering [Max 1995] [S2]. This produces a raw neural rendering RGB image $\mathbf{I}_{128} \in \mathbb{R}^{3 \times 128 \times 128}$ and a feature image $\mathbf{I}_f \in \mathbb{R}^{32 \times 128 \times 128}$ [S2]. The first three channels of $\mathbf{I}_f$ correspond to $\mathbf{I}_{128}$ [S2].
6.  **Depth Extraction:** Extract a dense depth map $\mathbf{I}_D \in \mathbb{R}^{128 \times 128}$ from the volume rendering [S2].
7.  **Super-Resolution:** Feed the feature image $\mathbf{I}_f$ and the raw RGB image $\mathbf{I}_{128}$ into a 2D super-resolution network to yield the final output $\mathbf{I}_{512} \in \mathbb{R}^{3 \times 512 \times 512}$ [S2]:
    $$
    \text{SuperRes}(\mathbf{I}_f, \mathbf{I}_{128}) = \mathbf{I}_{512}
    $$

**Parameters**
*   **Image Resolution:** Input images and final output are $512 \times 512$ [S2].
*   **Intermediate Resolution:** Neural rendering occurs at $128 \times 128$ [S2].

**Pitfalls**
*   **Axis-Aligned Bias:** The orthogonal coordinate transformation can be challenging to handle complicated structures due to the axis-aligned bias of the representation [S1, unofficial].
*   **Local Minima:** Optimization is more easily affected by local minima compared to MLP representations, which benefit from inductive smoothness bias [S1, unofficial].
*   **Memory:** While efficient compared to 3D grids, it still requires high spatial resolution to represent fine details, leading to a large memory footprint relative to sparse methods [S1, unofficial].

## Operating characteristics
*   **Performance:** The efficient design allows rendering from a triplane at 42 fps on an RTX 3090 [S2].
*   **Quality:** Provides quality comparable to state-of-the-art 2D GANs in terms of Fréchet Inception Distance (FID) [S2].
*   **Limitations:** The axis-aligned nature limits its ability to model complex, non-axis-aligned structures without high resolution [S1, unofficial].

## Background
The EG3D Triplane Representation was introduced in EG3D [Chan et al. 2022] as a method for learning unconditional 3D-aware image generation from single-view images [S2]. It builds upon the ConvONet approach [43] which first utilized a tri-plane representation to model 3D scenes [S1, unofficial]. In the context of Factor Fields, this representation corresponds to $N=1$, with an orthogonal 2D coordinate transformation $\gamma_1(\mathbf{x})$, 2D map basis $f_1(\mathbf{x})$, and MLP projection $P(\mathbf{x})$ [S1, unofficial].

## Sources
[S1] Factor Fields: A Unified Framework for Neural Fields and Beyond, https://doi.org/10.48550/arxiv.2302.01226, fetched 2026-10-26 (unofficial)
[S2] Real-Time Radiance Fields for Single-Image Portrait View Synthesis, https://doi.org/10.1145/3592460, fetched 2026-10-27
