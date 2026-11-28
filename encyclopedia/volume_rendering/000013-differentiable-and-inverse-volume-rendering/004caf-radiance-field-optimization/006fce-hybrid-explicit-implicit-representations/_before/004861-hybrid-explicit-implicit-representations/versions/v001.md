# Hybrid Explicit-Implicit Representations

Also called: Hybrid Radiance Fields, Explicit-Implicit Hybrid Representations

## Overview
Hybrid Explicit-Implicit Representations address the trade-off between the memory efficiency of explicit structures and the continuous flexibility of implicit neural decoders in radiance field optimization. Unlike pure implicit methods that suffer from high query latency due to MLP inference, or pure explicit methods that lack continuous representation capabilities, hybrid approaches store features in structured formats (e.g., voxel grids, hash tables) and decode them via lightweight networks [S1]. This family balances memory efficiency and rendering speed by leveraging cheap lookups for feature retrieval while maintaining the ability to model complex scene geometry and appearance [S1].

## Shared principles
The core mechanism involves two components: an **Explicit Feature Storage** ($V$) and a **Decoder** ($D$).

1.  **Explicit Feature Storage ($V$):** A data structure holding feature vectors. Common implementations include:
    *   **Voxel Grid:** A 3D array where each cell stores a feature vector.
    *   **Hash Table:** A mapping from spatial coordinates to feature vectors (e.g., Neural Grids [S1]).
    *   **Basis Vectors/Matrices:** A set of learned basis functions.

2.  **Decoder ($D$):** A function (typically a Multi-Layer Perceptron (MLP) or Spherical Harmonics) that maps the retrieved feature vector $v_p$ to physical properties [S1].

**Algorithm Steps:**
1.  **Initialize Storage:** Construct the explicit feature storage $V$. If using a voxel grid, initialize the grid dimensions and feature dimensions. If using a hash table, initialize the hash function and capacity [S1].
2.  **Feature Retrieval:** For a query point $p = (x, y, z)$ in 3D space, retrieve the corresponding feature vector $v_p$ from $V$.
    *   If $V$ is a voxel grid, perform tri-linear interpolation of the surrounding voxel features to obtain $v_p$ [S1].
    *   If $V$ is a hash table, compute the hash of $p$ and retrieve the associated feature vector.
3.  **Decoding:** Pass the feature vector $v_p$ through the decoder $D$ to predict the volume density $\sigma(p)$ and color $c(p, d)$, where $d$ is the viewing direction [S1].
    *   $\sigma(p) = D_\sigma(v_p)$
    *   $c(p, d) = D_c(v_p, d)$
    *   The decoder $D$ can be an MLP [S1] or Spherical Harmonics [S1].
4.  **Rendering:** Use standard volume rendering integration along rays passing through the scene, summing the contributions of $\sigma$ and $c$ weighted by transmittance.

**Parameters:**
*   **Storage Type:** Voxel grid, hash table, or basis vectors [S1].
*   **Decoder Type:** MLP or Spherical Harmonics [S1].
*   **Interpolation Method:** Tri-linear interpolation is standard for voxel grids [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Tri-linear interpolation assumes continuous variation between voxels; sharp discontinuities may require higher resolution or specialized handling [S1].
*   **Boundaries:** Points outside the defined storage domain must be handled (e.g., clamping, zero-padding, or out-of-bounds flags) [G].
*   **Degenerate Input:** Empty regions in the scene may result in sparse feature storage, requiring efficient sparse data structures or padding strategies [S1].

## Subtopics

### [Hybrid Representation](7846-hybrid-representation.md)
A general framework combining explicit feature storage with an implicit decoder to balance memory efficiency and rendering speed.

### [DVGO (Deep Voxel Grids)](12077-dvgo-deep-voxel-grids.md)
Uses explicit voxel grids for density inference and implicit MLPs for color inference to achieve real-time performance.

### MERF
*   **Definition:** MERF is a volumetric scene representation that achieves real-time rendering of large-scale scenes in a browser by combining a sparse feature grid and high-resolution 2D feature planes, with a lossless baking procedure to convert the optimized parameterization into a fast rendering format [S1]. It uses a unified radiance field representation with two different underlying parameterizations: one for optimization (NGP hash grid) and one for rendering (baked binary occupancy grid and triplanes) [S1].
*   **Steps:** During optimization, MERF uses the NGP hash grid structure to compress the parameterization, allowing for differentiable sparsification and providing an inductive bias that aids convergence [S1]. After optimization, it queries the recovered NGP to explicitly bake out the MERF and create a binary occupancy grid to accelerate rendering [S1].
*   **Performance:** MERF achieves better image quality for the same amount of compression as SNeRG++, matching the output of a SNeRG++ model that is over 17x larger [S1]. Compared to TensoRF, MERF speeds up query time by removing the need for matrix multiplication and halves memory bandwidth consumption [S1].

## Background
Hybrid representations emerged to address the limitations of pure Neural Radiance Fields (NeRF), which are compute-intensive for real-time rendering, and pure explicit methods, which lack the flexibility of continuous representations. The concept of storing features in a structured format (voxel grids, hash tables) and decoding them via lightweight networks was introduced to balance memory efficiency and rendering speed [S1]. Notable implementations include DVGO, which separates density and color inference into explicit and implicit components respectively, and MERF, which introduces a baking procedure to transition from an optimized hash-grid representation to a real-time renderable format [S1][S2].

## Sources
[S1] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
