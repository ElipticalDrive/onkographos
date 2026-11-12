# City-on-Web Baking

Also called: City-on-Web Scene Representation & Baking

## Overview
City-on-Web Baking is a post-training procedure that converts a neural field representation into a block-sparse, texture-based format optimized for real-time web rendering. It solves the problem of storing and transmitting large-scale neural scenes by baking high-resolution 2D planes and low-resolution 3D voxel grids per block, storing only non-empty voxels. It sets itself apart from its parent, MERF Baking, by explicitly targeting web-based delivery through texture compression and block-sparse storage, trading higher frame rates for neural field reconstruction capabilities compared to mesh rasterization methods.

## How it works
**Inputs and Outputs:**
*   **Input:** A trained neural radiance field (NeRF) or neural rendering model, typically using multi-resolution hash encoding and MLPs for density, color, and specular features [S1].
*   **Output:** A set of compressed PNG texture assets representing baked grid attributes, organized in a block-sparse structure with Level of Detail (LOD) support [S1].

**Data Structures:**
*   **Block-Sparse 3D Voxel Grid:** Stores only non-empty voxels. Each block has a voxel resolution of $512^3$ [S1].
*   **High-Resolution 2D Planes (Triplanes):** Stored as images within each block. Resolution is $2048^2$ per plane [S1].
*   **Binary Occupancy Grids ($\phi^{k,l}$):** Identifies occupied voxels within each block [S1].
*   **Texture Assets:** Baked grid attributes stored as images to reduce memory footprint and accelerate web transmission [S1].

**Steps:**
1.  **Ray Sampling:** Render all training rays to collect ray samples initially [S1].
2.  **Thresholding:** Retain samples with opacity and weight values above a certain threshold; discard samples below the threshold [S1].
3.  **Occupancy Marking:** Use the preserved samples to mark the adjacent eight grid points as occupied in the binary occupancy grids $\phi^{k,l}$ [S1].
4.  **Baking:** Bake high-resolution 2D planes (triplanes) and a low-resolution 3D voxel grid within each block to derive the attribute function $f^{k,l}$ [S1].
5.  **Sparse Storage:** Store only the non-empty 3D voxels using a block-sparse format [S1].
6.  **LOD Generation:** Downsample the occupancy grid using max-pooling to generate lower-resolution LODs for efficient rendering [S1].
7.  **Compression:** Compress the resulting textures into PNG format to reduce storage footprint [S1].

**Parameters:**
*   Voxel resolution per block: $512^3$ [S1].
*   Triplane resolution per block: $2048^2$ [S1].
*   Encoder: 4-layer MLP with 64 hidden dimensions (after multi-resolution hash encoding) [S1].
*   Deferred MLP: 3-layer MLP with 16 hidden dimensions for residual view-dependent color [S1].
*   Batch size: 16,384 rays per batch [S1].

**Edge Cases and Pitfalls:**
*   **Threshold Selection:** The threshold for retaining samples is critical; too low includes noise, too high removes valid geometry. The source mentions "a certain threshold" but does not specify the exact value [S1].
*   **Web Performance:** The method results in lower frame rates compared to mesh rasterization-based methods due to the use of volume rendering [S1].
*   **Storage vs. Quality:** While PNG compression reduces storage, it may introduce artifacts. The method maintains high-fidelity results even at the lowest LOD, but resource usage decreases with higher LODs [S1].

## Subtopics

### Block-Sparse Triplane and Voxel Baking
This variant emphasizes the specific mechanism of marking occupancy and baking. It renders training rays, retains high-opacity/weight samples, marks adjacent grid points in $\phi^{k,l}$, and bakes triplanes and voxel grids. It explicitly stores non-empty voxels in a block-sparse format and downsamples occupancy grids via max-pooling for LODs, compressing textures to PNG [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Significantly reduced compared to dense grids due to block-sparse storage and texture compression. The lowest LOD still maintains high-fidelity results, but resource usage increases with higher resolution blocks [S1].
*   **Measured Performance:** On the Matrix City, Campus, and Rubble datasets, the method achieves competitive PSNR, LPIPS, and SSIM scores (e.g., 25.87 PSNR on Matrix City) [S1].
*   **Scaling:** The method is designed for large-scale scenes (e.g., 1200 × 800 square meters) and supports LODs to manage resource loading on web platforms [S1].
*   **Strengths:** Enables neural field reconstruction for large-scale scenes on the web; reduces memory and transmission costs via compression and sparse storage [S1].
*   **Limitations:** Lower frame rates compared to mesh rasterization methods due to volume rendering overhead [S1].
*   **When to use:** Real-time neural rendering of large-scale scenes on web platforms where neural field quality is prioritized over maximum frame rate [S1].
*   **When not to use:** Applications requiring mesh rasterization performance or where neural field reconstruction is unnecessary [S1].

## Background
City-on-Web was introduced in 2024 by F. Author et al. in the paper "City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web" [S1]. It builds on MERF Baking and is implemented using the nerfstudio framework with the tiny-cuda-nn extension, targeting JavaScript web applications with GLSL rendering [S1].

## Sources
[S1] City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web, https://doi.org/10.1007/978-3-031-72970-6_22, fetched 2026-10-26
