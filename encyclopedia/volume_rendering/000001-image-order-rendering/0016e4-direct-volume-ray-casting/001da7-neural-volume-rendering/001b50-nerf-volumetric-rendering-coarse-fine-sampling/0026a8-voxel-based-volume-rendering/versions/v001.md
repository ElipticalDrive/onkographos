# Voxel-based Volume Rendering

Also called: Voxel-based Radiance Field, Dense Voxel Rendering

## Overview
Voxel-based Volume Rendering is a technique that trains a dense or semi-dense voxel representation of a radiance field, where multiple voxels contribute to a single sample during rendering [S1, unofficial]. It differs from sparse voxelization (such as octrees) by using a fixed grid structure where the model determines opacity and color through aggregation over voxel intervals [S1, unofficial]. It is primarily used as a coarse model in two-tier sampling architectures to identify opaque regions in space before fine-grained sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of calibrated images with known camera poses $(P_i)$ and intrinsics.
*   **Output:** A rendered RGB image and optionally depth/opacity maps for a target camera pose.

**Data Structures:**
*   **Voxel Grid:** A 3D tensor of shape $D \times D \times D$ (where $D$ is the resolution, e.g., 128).
*   **Voxel Values:** Each voxel stores a feature vector. In standard implementations, this includes density ($\sigma$) and color ($C$) or a latent feature vector decoded by an MLP [S1, unofficial]. The opacity is predicted without view direction to encourage structural consistency [S1, unofficial].

**Steps:**
1.  **Grid Initialization:** Initialize a 3D grid of size $D^3$. Each cell contains learnable parameters (density and color/feature).
2.  **Ray Casting:** For each pixel in the target image, cast a ray from the camera origin through the pixel center into the volume.
3.  **Voxel Intersection:** Determine the entry and exit points of the ray within the voxel grid boundaries.
4.  **Sampling:** Sample points along the ray. Unlike continuous NeRF sampling, samples are often aligned with voxel boundaries or distributed uniformly within the ray's intersection with the grid [S1, unofficial].
5.  **Aggregation:** For each sample point, retrieve the voxel value. If a sample falls within a voxel, the contribution is derived from that voxel's stored density and color. Multiple voxels may contribute to a single sample interval if the sampling strategy spans voxel boundaries [S1, unofficial].
6.  **Volume Rendering:** Perform standard volumetric rendering integration (alpha compositing) along the ray:
    $$
    C(r) = \sum_{i=1}^{N} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) c_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance, $\sigma_i$ is the density, $\delta_i$ is the distance between samples, and $c_i$ is the color [S1, unofficial].
7.  **Training:** Minimize the difference between rendered and ground truth images using gradient descent.

**Parameters:**
*   **Grid Resolution ($D$):** Typical values include 128, 256, or 512. Higher resolution increases memory usage quadratically/cubically but improves detail [S1, unofficial].
*   **Number of Samples:** Determined by the grid resolution and ray length.

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Dense voxel grids consume significant memory ($O(D^3)$). For $D=128$, this is manageable; for $D=512$, it becomes prohibitive without optimization [S1, unofficial].
*   **Aliasing:** Low-resolution grids may produce blurry results or aliasing artifacts if not properly filtered [S1, unofficial].
*   **Boundary Conditions:** Rays exiting the grid must be handled to avoid undefined behavior.

## Subtopics

### [Coarse Ray Renderer](11371-coarse-ray-renderer.md)
The Coarse Ray Renderer (CRR) is a fast rendering technique that provides coarse density estimation and ray features for downstream processes in neural volume rendering, specifically serving the Fine Voxel Renderer (FVR) and Progressive Blur Estimation (PBE). It differs from standard per-voxel ray casting by aggregating features along a ray and inferring color once per ray rather than querying every voxel, which significantly reduces computational cost at the expense of per-voxel accuracy. It builds on Voxel-based Volume Rendering by utilizing a decomposed explicit feature volume.

### Deterministic Integration for Volume Rendering (DIVeR)
DIVeR is a method that jointly optimizes a feature voxel grid and a decoder MLP while applying sparsity regularization and voxel culling [S2]. It innovates the volume rendering process by performing deterministic ray sampling on the voxel grid [S2]. Deterministic ray sampling produces an integrated feature for each ray interval defined by the intersection of the ray with a specific voxel [S2]. This integrated feature is then decoded by an MLP to produce the density and color of the ray interval [S2]. DIVeR effectively reverses the usual order of volume sampling and MLP evaluation found in NeRF methods [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot D^3)$ for training, where $N$ is the number of pixels and $D$ is the grid resolution. Rendering is $O(N \cdot S)$, where $S$ is the number of samples per ray.
*   **Memory Complexity:** $O(D^3)$ for the voxel grid. This is a significant limitation compared to sparse representations.
*   **Performance:** Fast training and inference compared to continuous MLP-based NeRFs for low-to-medium resolutions. However, memory constraints limit scalability to high resolutions [S1, unofficial].
*   **Strengths:** Structurally consistent opacity prediction; efficient for coarse modeling; simple implementation.
*   **Limitations:** High memory usage for high resolutions; potential for aliasing; less flexible than sparse representations for complex scenes.
*   **When to Use:** As a coarse model in two-tier sampling; scenes with moderate complexity; resource-constrained environments where memory is not the primary bottleneck.
*   **When Not to Use:** High-resolution requirements; highly sparse scenes; real-time applications requiring very high fidelity.

## Background
Voxel-based volume rendering in the context of neural radiance fields was introduced as a baseline technique in educational materials for Fourier Feature Networks and Neural Volume Rendering [S1, unofficial]. It serves as a conceptual stepping stone to more advanced methods like NeRF, highlighting the trade-offs between discrete and continuous representations. The technique was later refined in methods like DIVeR, which introduced deterministic integration and feature decoding [S2].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
