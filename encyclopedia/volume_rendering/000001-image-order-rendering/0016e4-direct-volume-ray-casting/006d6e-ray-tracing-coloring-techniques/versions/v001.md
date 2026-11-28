# Ray-tracing coloring techniques

Also called: Coloring strategies in ray casting, Scalar vs Color compositing

## Overview
Ray-tracing coloring techniques define the order of operations for mapping volumetric data to visual output, specifically distinguishing between combining scalar values before color mapping and combining color values during ray traversal [S1]. This distinction determines whether the integration process operates on a single intensity channel or full RGB channels, impacting both the physical accuracy of light attenuation and the visual interpretation of spectral data [S1].

## How it works
The method consists of two distinct strategies for handling the transfer function and compositing steps defined in [Direct volume ray casting](19-direct-volume-ray-casting.md).

### Inputs and Outputs
*   **Input**: A volumetric dataset $V$ (scalar or spectral), a viewing ray $r(t)$, and a transfer function $T$.
*   **Output**: A final pixel color $\mathbf{C}_{out}$.

### Strategy 1: Scalar Combination (Greyscale/Post-mapping)
In this approach, the ray integration is performed on scalar values first. The color is applied only after the integration is complete.

1.  **Sampling**: For each sample $i$ along the ray, retrieve the scalar value $V_i$ [S1].
2.  **Accumulation**: Combine the scalar values encountered by the ray. The source notes indicate a simple combination of scalar values, though standard volume rendering typically uses opacity-weighted accumulation. The resulting combined scalar $S_{combined}$ is computed [S1].
3.  **Color Mapping**: Map the final combined scalar $S_{combined}$ to a color $\mathbf{C}_{out}$ using a lookup table or transfer function [S1].
4.  **Display**: Render $\mathbf{C}_{out}$ as greyscale or mapped color [S1].

### Strategy 2: Color Combination (Pre-mapping/In-situ)
In this approach, the transfer function is applied to each sample individually before compositing.

1.  **Sampling**: For each sample $i$ along the ray, retrieve the data value $V_i$ [S1].
2.  **Pre-mapping**: Map $V_i$ to a full color vector $\mathbf{c}_i$ and opacity $\alpha_i$ using the transfer function $T(V_i) \rightarrow (\mathbf{c}_i, \alpha_i)$ [S1].
3.  **Color Compositing**: Combine the full color information as the ray is traced. This follows the standard front-to-back compositing equation defined in the parent article:
    $$
    \mathbf{C}_{out} = \sum_{i=1}^{N} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    [S1].

### Parameters and Edge Cases
*   **Transparency**: The transparency of a pixel can vary from fully opaque to fully transparent, which is controlled by the opacity component $\alpha$ in Strategy 2 [S1].
*   **Spectral Data**: Strategy 2 is particularly relevant for visualizing spectral cubes (e.g., astronomical data), where mapping data to color before combination preserves spectral information that would be lost in scalar combination [S1].

## Operating characteristics
*   **Strategy 1 (Scalar)**:
    *   **Performance**: Potentially faster if the final color mapping is a simple lookup, as compositing involves only scalar addition/multiplication.
    *   **Visuals**: May produce inaccurate lighting effects because attenuation is not calculated per-channel. Suitable for visualizing intensity distributions where color is purely categorical or intensity-based [S1].
*   **Strategy 2 (Color)**:
    *   **Performance**: Higher computational cost per sample due to vector operations (RGB) during compositing.
    *   **Visuals**: Physically more accurate for multi-spectral data. Correctly handles the interaction of different colors as they attenuate through the volume [S1].

## Background
These techniques are fundamental to volume visualization, with early implementations appearing in software suites like Karma (Gooch 1996) which utilized texture-based volume rendering [S1]. Modern applications include tools like 3D Slicer and astronomical visualization of spectral cubes (e.g., THINGS survey data), where the choice between scalar and color compositing affects the interpretation of complex data [S1].

## Sources
[S1] Real-time colouring and filtering with graphics shaders, https://doi.org/10.1093/mnras/stx1676, fetched 2026-10-26
