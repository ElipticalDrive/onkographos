# CoDeF
Also called: Computational Deformation Field
## Overview
CoDeF (Computational Deformation Field) is a deformation representation used within Neural and primitive-based representations for dynamic volume rendering, specifically in the context of 4D Gaussian Splatting. It models temporal motion by mapping points from a canonical space to a deformed space using a multiresolution hash table. It sets itself apart from sibling methods like [Multi-MLP Linear Layer](15116-multi-mlp-linear-layer.md) by relying on hash-grid-based deformation rather than explicit MLP parameterizations or latent code conditioning, prioritizing storage efficiency and compatibility with standard image processing pipelines.
## How it works
CoDeF operates by maintaining a canonical representation of the scene (typically a static Gaussian field or texture) and a temporal deformation field. The deformation field maps coordinates from the canonical frame to the target frame at time $t$.

**Inputs and Outputs:**
*   **Input:** Canonical coordinates $\mathbf{x}_c$, time $t$, and potentially velocity or optical flow features.
*   **Output:** Deformed coordinates $\mathbf{x}_t$ and/or deformation vectors $\Delta \mathbf{x}$.

**Data Structures:**
The core data structure is a multiresolution hash table (hash grid).
*   **Hash Grid:** A sparse hash map storing deformation vectors (typically 3D floats) at various resolutions.
*   **Hash Function:** Maps 3D coordinates $(x, y, z)$ and time $t$ to a hash index.
*   **Interpolation:** Trilinear interpolation is used between hash entries to ensure smooth deformation fields.

**Steps:**
1.  **Canonical Representation:** Store the static scene properties (color, opacity, Gaussian parameters) in a canonical space. This allows the application of standard 2D image processing algorithms [S1].
2.  **Deformation Query:** For a given point $\mathbf{x}_c$ at time $t$:
    a. Compute the hash index for $(\mathbf{x}_c, t)$ using the multiresolution hash table.
    b. Retrieve the stored deformation vector $\mathbf{v}$ from the hash grid.
    c. Interpolate $\mathbf{v}$ if necessary (trilinear interpolation across neighboring grid cells).
3.  **Coordinate Transformation:** Apply the deformation to the canonical coordinate:
    $$ \mathbf{x}_t = \mathbf{x}_c + \mathbf{v} $$
    Or, if the hash table stores absolute positions:
    $$ \mathbf{x}_t = \mathbf{v} $$
4.  **Rendering:** Render the deformed point $\mathbf{x}_t$ using standard volume rendering or splatting techniques.
5.  **Temporal Propagation:** Results from image processing applied to the canonical image are propagated across time via the temporal deformation field [S1].

**Parameters:**
*   **Hash Grid Resolution:** Determines the granularity of the deformation field. Higher resolution captures complex motion but increases memory.
*   **Number of Hash Tiers:** Multiresolution hashing uses multiple tiers of resolution.
*   **CCA Profile:** CoDeF is often used in conjunction with Point-Cloud Compression (CCA). The CCA profile (Low, Mid, Default) affects the bitrate and storage of the Gaussian attributes, while CoDeF handles the deformation [S2].

**Edge Cases and Pitfalls:**
*   **Optical Flow Accuracy:** CoDeF is limited by the accuracy of the underlying optical flow or motion estimation. Inaccurate flow leads to artifacts in the deformation field [S1].
*   **Complex Motion:** It has difficulty modeling complex, non-rigid, or high-frequency motion that exceeds the representational capacity of the hash grid [S1].
*   **Boundary Conditions:** Proper handling of points outside the hash grid bounds is required (e.g., clamping or zero deformation).

## Subtopics

### CDF Capacity Sweeps
CDF Capacity Sweeps refer to the systematic variation of the deformation network/grid capacity to generate rate-distortion operating points, evaluating storage-quality trade-offs independently of CCA compression [S2]. This variant is used to analyze the efficiency of the CoDeF representation.

*   **Definition:** Systematically varies the deformation network/grid capacity to generate rate-distortion operating points, evaluating storage-quality trade-offs independently of CCA compression [S2].
*   **Parameters:**
    *   **CDF Capacity Tiers:** Small, Mid, Default. Controls deformation module size and representational capacity [S2].
    *   **Hash-Grid Capacity Tiers (for baseline comparison):** Hash-Small $\equiv H^*-4$, Hash-Mid $\equiv H^*-2$, Hash-Default $\equiv H^*$, where $H^*$ is scene-specific default [S2].
*   **Performance Numbers:**
    *   CC-4DGS configurations stay under 30 MB total and under 2 MB Deform-module storage across all tiers, spanning $\approx 30.4–32.0$ dB PSNR [S2].
    *   Swift4D spans 31–94 MB total and 5–68 MB Deform-module storage for the same PSNR range [S2].
*   **Comparison:** CC-4DGS achieves comparable PSNR to Swift4D's default configuration with an order-of-magnitude smaller storage envelope. Even Swift4D's most aggressively shrunk hash-grid uses more Deform storage than CC-4DGS at full capacity, confirming that the storage advantage is not merely due to point-cloud compression but also reflects the compact CDF representation [S2].

## Operating characteristics
*   **Time Complexity:** Querying the hash grid is $O(1)$ on average, making it efficient for real-time rendering. Interpolation adds a small constant factor.
*   **Memory Complexity:** Depends on the hash grid size and number of tiers. CoDeF is designed to be storage-efficient, with deformation module storage often under 2 MB for typical scenes [S2].
*   **Measured Performance:** On N3DV scenes, CC-4DGS with CoDeF achieves 30.4–32.0 dB PSNR with total model size under 30 MB [S2].
*   **Strengths:** High storage efficiency, compatibility with standard image processing pipelines, and efficient temporal propagation [S1].
*   **Limitations:** Limited by optical flow accuracy and difficulty in modeling complex motion [S1].
*   **When to Use:** When storage efficiency is critical and motion is relatively smooth or predictable.
*   **When Not to Use:** For scenes with highly complex, non-rigid, or high-frequency motion where optical flow estimation is unreliable [S1].

## Background
CoDeF was introduced in the context of dynamic Gaussian Splatting, specifically in the CC-4DGS method [S2]. It builds on the idea of using multiresolution hash tables for efficient spatial and temporal representation [S1]. The method is part of a broader trend in 3D editing and dynamic scene representation, leveraging NeRF and 3DGS techniques [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] CC-4DGS: Computational Deformation and Point-Cloud Compression for Storage-Efficient Dynamic Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3721136, fetched 2026-10-27
