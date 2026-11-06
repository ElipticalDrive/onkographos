# Supplementary Feature Input for Super-Resolution
Also called: Multi-Channel Input for SR Network

## Overview
A technique where the super-resolution network accepts multiple feature maps alongside the low-resolution color image to improve reconstruction quality, specifically adapted for volumetric rendering [S1]. It sets itself apart from standard image super-resolution by incorporating volumetric-specific auxiliary data (depth, color-opacity vectors) to resolve ambiguities inherent in volume rendering, such as undefined depth in transparent regions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ feature maps concatenated along the channel dimension.
*   **Output:** A single high-resolution RGB image.

**Data Structures:**
The input tensor has shape $(H, W, C_{total})$, where $H$ and $W$ are the spatial dimensions and $C_{total}$ is the sum of channels from all supplementary features. The specific layout of the input channels is:
1.  **Final Rendered Color:** 3 channels (RGB).
2.  **Depth Map:** 1 channel.
3.  **Color-Opacity Vector:** 4 channels (RGBA). The opacity channel is included in the vector but may not be visually distinct or used directly in the final output generation, yet it is part of the input feature set [S1].
4.  **Motion Vector Image:** 2 channels (typically $u, v$ displacement components) [S1].

**Numbered Steps:**
1.  **Render Auxiliary Features:** For each frame, render the standard RGB volume rendering. Simultaneously, compute and store:
    *   The depth value at the position of maximum alpha along each ray. This is chosen because the final rendering contribution is dominated by this point, providing a more stable depth signal than average depth [S1].
    *   The color and opacity values at this same maximum-alpha position.
    *   Motion vectors between consecutive frames (if temporal processing is enabled) [S1].
2.  **Concatenate Features:** Stack the RGB image, depth map, color-opacity vector, and motion vectors into a single multi-channel tensor.
3.  **Process via CNN:** Pass the concatenated tensor through the CNN Post-Processing Network [7179-cnn-post-processing-network.md]. This network typically consists of a shallow CNN with a feed-forward perceptron layer to convert feature space to RGB space [7179-cnn-post-processing-network.md].
4.  **Output Reconstruction:** The network outputs the super-resolved RGB image.

**Formulas:**
Let $I_{LR}$ be the low-resolution RGB image. Let $D$ be the depth map at max alpha. Let $C_{op}$ be the color-opacity vector at max alpha. Let $M$ be the motion vector field.
The input to the SR network $F_{in}$ is:
$$ F_{in} = \text{Concat}(I_{LR}, D, C_{op}, M) $$
The output $I_{HR}$ is:
$$ I_{HR} = \text{SR\_Network}(F_{in}) $$

**Parameters:**
*   **Neighborhood Clamping:** A $3 \times 3$ neighborhood clamping is applied, which produced reasonably effective results for the volume rendering case [S1].

**Edge Cases and Pitfalls:**
*   **Depth Ambiguity:** Standard depth is not well-defined for volumetric data. Using depth at maximum alpha mitigates this by selecting the most visually significant surface [S1].
*   **Opacity Visibility:** The opacity channel in the color-opacity vector is not shown in visualizations but is part of the input structure [S1].

## Subtopics

### [Volumetric Super-Resolution with Auxiliary Features and Temporal Warping](7373-volumetric-super-resolution-with-auxiliary-features-and-temp.md)
Extends this method by adding temporal warping and a reweighting network to handle motion and lighting changes across frames.

## Operating characteristics
*   **Time Complexity:** Depends on the CNN architecture defined in [7179-cnn-post-processing-network.md]. The addition of supplementary features increases input channel count but does not inherently change the asymptotic complexity of the convolutional layers.
*   **Memory Complexity:** Increases linearly with the number of supplementary feature maps. Requires storing depth, color-opacity, and motion vectors per frame.
*   **Strengths:** Improves reconstruction quality by providing geometric and volumetric context that RGB alone lacks [S1].
*   **Limitations:** Requires rendering additional auxiliary buffers, increasing rendering time and memory bandwidth. Depth is still an approximation for complex volumetric structures [S1].

## Background
Originated in the context of medical volume visualization using direct volume rendering [S1]. It builds on previous works showing that reconstruction networks for surface data benefit from depth inputs [S1, ref 21, 14]. It adapts these surface-based insights to volumetric data by using max-alpha depth as a proxy for surface depth [S1].

## Sources
[S1] Deep Learning Based Super-Resolution for Medical Volume Visualization with Direct Volume Rendering, https://doi.org/10.1007/978-3-031-20713-6_8, fetched 2026-10-26
