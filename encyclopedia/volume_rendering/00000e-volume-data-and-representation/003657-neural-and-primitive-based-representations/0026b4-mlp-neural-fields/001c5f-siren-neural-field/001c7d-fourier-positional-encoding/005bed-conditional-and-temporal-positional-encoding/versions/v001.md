# Conditional and Temporal Positional Encoding

Also called: Conditional Positional Encoding (CPE), Coordinate-Time Encoding, One-Blob Encoding

## Overview
This family extends standard positional encoding to handle dynamic scenes and external conditioning signals. It addresses the limitation of static encodings by introducing time-dependent phase shifts or explicit spatiotemporal concatenation, enabling neural fields to represent temporal evolution or conditional attributes (e.g., text prompts) without re-optimizing latent codes per sample. It differs from its parent [Fourier Positional Encoding](1267-fourier-positional-encoding.md) by incorporating temporal indices or conditional latent vectors directly into the encoding mechanism or input concatenation, rather than relying solely on fixed frequency bands for spatial coordinates.

## Shared principles
The core principle is the augmentation of the input representation space to include non-spatial dimensions (time, view, or latent codes) or to modulate the encoding function itself based on these dimensions.

1.  **Input Augmentation**: Inputs are extended beyond static 3D coordinates $(x, y, z)$. This includes adding a temporal coordinate $\tau$ or view direction $(\theta, \phi)$, or concatenating latent vectors $z$ (shape/appearance) to the encoded spatial features.
2.  **Modulation**: The encoding function $\Gamma$ is no longer static. It may depend on a time index $t$ or a displacement field $\Delta p$, altering the phase or value of the basis functions.
3.  **Concatenation Strategy**: Encoded features are concatenated with latent codes or other modalities before entering the MLP. The order and placement (input-only vs. per-layer) are specific to the variant.

## Subtopics

### [Coordinate-Time Encoding](5105-coordinate-time-encoding.md)
Also known as One-Blob Encoding.
*   **What it is**: A continuous, single-scale positional encoding that maps a scalar coordinate $s$ to a vector $\mathbf{b}(s) \in \mathbb{R}^k$ using a Gaussian kernel. It serves as a continuous variant of one-hot encoding, rasterizing a kernel to represent coordinates.
*   **Difference**: Unlike the multi-frequency approach of [Fourier Positional Encoding](1267-fourier-positional-encoding.md), it provides a localized representation. It is single-scale, meaning it cannot represent high-frequency details as effectively as multi-scale encodings if features are smaller than the kernel's effective support.
*   **When to choose**: Use when handling dynamic conditions in in vivo sequences or bounded domains where localized, single-scale representation is preferred over global frequency decomposition.
*   **Link**: [Coordinate-Time Encoding](5105-coordinate-time-encoding.md)

### [Conditional NeRF with Sinusoidal Positional Encoding](16765-conditional-nerf-with-sinusoidal-positional-encoding.md)
*   **What it is**: A neural radiance field formulation that maps spatial position, view direction, and latent codes (derived from text prompts) to volumetric density and view-dependent color. It uses a feed-forward approach to update condition codes rather than optimizing them per sample.
*   **Difference**: Distinguished from standard NeRF by explicit conditioning on latent vectors and the use of a deformation network with $\tanh$-constrained displacements to handle global volume field manipulations.
*   **When to choose**: Use for text-driven 3D scene generation or manipulation where global shape changes are required.
*   **Link**: [Conditional NeRF with Sinusoidal Positional Encoding](16765-conditional-nerf-with-sinusoidal-positional-encoding.md)

#### Conditional Positional Encoding
*   **Definition**: A technique that applies phase shifts based on a time index to sampled features, inspired by Fourier transformation phase information retention [S1].
*   **Mechanism**:
    1.  Apply positional encoding to 6D attributes $(x, y, z, t, \theta, \phi)$ [S1].
    2.  Apply positional encoding to sampled features $F_{den}$ (density) and $F_{tex}$ (texture) [S1].
    3.  Explicitly assign phase shift to different frequencies using the time index $t$ [S1].
    4.  Calculate Conditional Positional Encoding (CPE) for texture features only, using the formula:
        $$
        \gamma(p_v) = \left( \sin\left(2\frac{\pi}{2^L} p_v + \frac{2\pi}{2^L} t\right), \cos\left(2\frac{\pi}{2^L} p_v + \frac{2\pi}{2^L} t\right) \right)
        $$
        where $p_v$ is the sampled feature vector, $L=5$ is the frequency order, and $t$ is the time index [S1].
*   **Pitfall**: CPE is **not** applied to density field features ($F_{den}$) to maintain zero initialization for correct volume rendering at the start of training; applying it would break this rule and cause learning collapse [S1].

## Background
The concept of conditional encoding in neural fields emerged to support dynamic novel view synthesis and text-driven generation. The specific "Conditional Positional Encoding" (CPE) with time-indexed phase shifts was introduced in V4D (Voxel for 4D Novel View Synthesis) to handle temporal dynamics in 4D scenes [S1]. Coordinate-Time Encoding (One-Blob) is noted for its application in in vivo sequence analysis, offering a localized alternative to Fourier methods . Conditional NeRF variants leverage sinusoidal encodings combined with latent codes to enable text-to-3D generation [S1].

## Sources
[S1] V4D: Voxel for 4D Novel View Synthesis, https://doi.org/10.1109/tvcg.2023.3312127, fetched 2026-10-26
[S2] Coordinate-Time Encoding details, unspecified source, fetched 2026-10-26
