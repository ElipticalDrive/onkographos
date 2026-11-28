# DyNeRF

Also called: Dynamic Neural Radiance Fields (baseline), Time-conditioned NeRF

## Overview
DyNeRF is a baseline method for representing volumetric videos by extending the standard Neural Radiance Field (NeRF) architecture to handle temporal dynamics [S1]. It solves the problem of novel view synthesis for dynamic scenes by conditioning a single shared MLP on both spatial coordinates and a temporal embedding [S1]. Unlike methods that decompose scenes into static and deforming components or use canonical space mappings, DyNeRF treats the entire volume as a time-varying function, predicting the radiance field for a specific video frame directly from its input coordinates [S1]. It differs from its sibling [D-NeRF](6569-point-nerf.md) by lacking an explicit deformation field or canonical space, relying instead on direct temporal conditioning [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial coordinate $\mathbf{x} = (x, y, z)$, a viewing direction $\mathbf{d} = (d_x, d_y, d_z)$, and a time embedding $t$ representing the specific video frame or timestamp [S1].
*   **Output:** Volume density $\sigma$ and view-dependent color $\mathbf{c}$ [S1].

**Data Structures:**
*   **MLP:** A single multi-layer perceptron (MLP) shared across all time steps and spatial locations [S1]. The exact architecture (number of layers, hidden dimensions) is not specified in the sources, but it is described as a "NeRF-like MLP" [S1].
*   **Time Embedding:** The time $t$ is encoded into a vector representation before being concatenated with the spatial and directional inputs [S1]. The specific encoding function (e.g., Fourier features, sinusoidal) is not detailed in the provided notes [G: Typically, NeRF variants use positional encoding for coordinates and potentially for time, but DyNeRF's specific time embedding mechanism is not explicitly defined in the sources].

**Algorithm Steps:**
1.  **Temporal Encoding:** Convert the scalar time value $t$ into a time embedding vector $\mathbf{e}_t$ [S1].
2.  **Input Concatenation:** Concatenate the spatial coordinate $\mathbf{x}$, viewing direction $\mathbf{d}$, and time embedding $\mathbf{e}_t$ into a single input vector [S1].
3.  **MLP Evaluation:** Pass the concatenated vector through the shared MLP [S1].
4.  **Output Prediction:** The MLP outputs the volume density $\sigma$ and the view-dependent color $\mathbf{c}$ for the ray at time $t$ [S1].
5.  **Rendering:** Use the standard volume rendering equation (as defined in the parent article) to integrate $\sigma$ and $\mathbf{c}$ along the ray to produce the final pixel color [S1][S5].

**Parameters:**
*   **MLP Architecture:** Not given in the sources.
*   **Time Embedding Dimension:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **High-Frequency Detail Loss:** The single shared MLP structure tends to lose details in high-frequency regions of the image [S1]. This is a known limitation compared to methods that use more specialized representations for different scene components [S1].
*   **Complex Motion:** While not explicitly detailed as a failure mode in the same way as D-NeRF, the direct conditioning on time may struggle with complex, non-rigid motions that are difficult to capture with a single monolithic network [S1].

## Subtopics

### [Decomposed Spatiotemporal Representation](7525-decomposed-spatiotemporal-representation.md)
Decomposed Spatiotemporal Representation is a method for modeling dynamic scenes by categorizing volumetric areas into static, deforming, and new content, assigning each category a specialized representation field to optimize for temporal consistency and computational efficiency. It differs from the parent DyNeRF method, which uses a single monolithic MLP conditioned on time, by explicitly separating the scene into distinct temporal behaviors, allowing static regions to remain time-invariant and deforming regions to use canonical space mappings.

### FDRF
FDRF is a method that uses time-aware voxel features to represent scenes and simulates rough motion trajectories with a small coordinate deformation network [S2]. It enhances time information in the radiation network and significantly accelerates the optimization of dynamic radiation fields while maintaining high rendering quality [S2]. It applies a multi-distance interpolation method to voxel features to simulate both small and large motions [S2].

## Operating characteristics
*   **Performance:** On the T-LESS dataset, DyNeRF achieves a PSNR of 30.87, SSIM of 0.943, and LPIPS of 0.118 [S1].
*   **Speed:** Rendering time is 5195 milliseconds [S1].
*   **Storage:** Storage requirement is 239 MB [S1].
*   **Strengths:** Simple implementation as a direct extension of static NeRF [S1].
*   **Limitations:** Loses details in high-frequency regions due to the single shared MLP structure [S1]. Rendering time is significantly higher than some other methods (e.g., 5195 ms vs. 73 ms for NV, though NV has lower quality) [S1].
*   **When to use:** As a baseline for comparison in dynamic scene reconstruction [S1].
*   **When not to use:** When high-frequency detail preservation is critical or when computational efficiency is a primary concern [S1].

## Background
DyNeRF was introduced as a baseline method for representing volumetric videos using a NeRF-like MLP that takes spatial coordinates and temporal embeddings as input [S1]. It is cited in comparative studies against other dynamic NeRF variants such as C-NeRF, D-NeRF, and Neural Volumes [S1]. The method is noted for its simplicity but also for its limitations in capturing high-frequency details [S1].

## Sources
[S1] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
