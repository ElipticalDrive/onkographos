# Hybrid Representation

Also called: Explicit-Implicit Hybrid

## Overview
Hybrid Representation is a framework for radiance field optimization that combines explicit feature storage with an implicit decoder to efficiently reconstruct and render scenes, addressing the computational expense of pure implicit NeRFs [S1]. It sets itself apart from siblings like DVGO by providing a general architectural pattern where spatial features are stored in a discrete structure (e.g., voxel grid, hash table) and decoded via a neural network, rather than relying solely on dense voxel grids or purely implicit MLPs [S1].

## How it works
**Inputs:**
*   A set of training images with known camera poses [G].
*   A scene bounding box defining the spatial domain [G].

**Outputs:**
*   A trained Explicit Feature Storage $V$ [G].
*   A trained Decoder $D$ [G].
*   A rendered image for a given query ray [G].

**Data Structures:**
1.  **Explicit Feature Storage ($V$):**
    *   **Voxel Grid:** A 3D array of dimensions $D_x \times D_y \times D_z$ where each cell stores a feature vector of dimension $F$ [S1].
    *   **Hash Table:** A mapping from spatial coordinates to feature vectors, requiring a hash function $H$ and a capacity $C$ [S1].
    *   **Basis Vectors:** A set of learned basis functions or matrices [S1].
2.  **Decoder ($D$):**
    *   Typically a Multi-Layer Perceptron (MLP) or Spherical Harmonics (SH) coefficients [S1].

**Algorithm Steps:**
1.  **Initialize Storage:** Construct the explicit feature storage $V$ (e.g., voxel grid, hash table, or basis vectors) [S1].
2.  **Feature Retrieval:** For a query point $p = (x, y, z)$ in 3D space, retrieve the feature vector $v_p$ from $V$ [S1]. If $V$ is a voxel grid, perform tri-linear interpolation of the surrounding voxel features to obtain $v_p$ [S1]. If $V$ is a hash table, compute the hash of $p$ and retrieve the associated feature vector [G].
3.  **Decoding:** Pass $v_p$ through the decoder $D$ to predict volume density $\sigma(p)$ and color $c(p, d)$, where $d$ is the viewing direction [S1]. The decoder $D$ can be an MLP [S1] or Spherical Harmonics [S1].
4.  **Rendering:** Use standard volume rendering integration along rays passing through the scene, summing the contributions of $\sigma$ and $c$ weighted by transmittance [G].

**Formulas:**
*   **Baking Implicit NeRF into Multi-resolution Voxel Mesh:** The scene space is divided into a three-level voxel grid based on adaptive resolution [S2]. The voxel side length is defined as $s_k = \frac{S}{2^k}$ where $S$ is the maximum side length of the scene and $k=0, 1, 2$ correspond to LOD0 (512³), LOD1 (256³), and LOD2 (128³) [S2]. Radiation field sampling collects $N_s = 8$ uniform sampling points $x_p$ within each voxel $V_{i,j,k}$ [S2]. The voxel properties are calculated through the pre-trained implicit NeRF model $F_\theta$ with Fourier encoding $\gamma(\cdot)$: [S2]
    $$ c_{i,j,k} = \frac{1}{N_s} \sum_{p=1}^{N_s} F_\theta(\gamma(x_p)) $$
    $$ \sigma_{i,j,k} = \frac{1}{N_s} \sum_{p=1}^{N_s} F_\theta(\gamma(x_p)) $$
    where $c_{i,j,k}$ and $\sigma_{i,j,k}$ are the averaged color and density for voxel $(i,j,k)$ [S2].

**Parameters:**
*   **Storage Type:** Voxel grid, hash table, or basis vectors [S1].
*   **Decoder Type:** MLP or Spherical Harmonics [S1].
*   **Interpolation Method:** Tri-linear interpolation is standard for voxel grids [S1].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Tri-linear interpolation is standard for voxel grids [S1].
*   **Boundaries:** Points outside the defined storage domain must be handled (e.g., clamping, zero-padding, or out-of-bounds flags) [G].
*   **Degenerate Input:** Empty regions in the scene may result in sparse feature storage, requiring efficient sparse data structures or padding strategies [S1].

## Subtopics

### HybridNeRF
HybridNeRF is a radiance field optimization method that combines explicit surface representations with implicit volumetric modeling to address computational expenses and enable efficient reconstruction and rendering [S1].

### Baking Implicit NeRF into Multi-resolution Voxel Mesh
A technique to replace the computational bottleneck of MLP inference in implicit NeRF by pre-calculating voxel properties (color and density) into a multi-resolution voxel grid structure [S2]. Spatial division divides scene space into a three-level voxel grid based on adaptive resolution [S2]. The voxel side length is defined as $s_k = \frac{S}{2^k}$ where $S$ is the maximum side length of the scene and $k=0, 1, 2$ correspond to LOD0 (512³), LOD1 (256³), and LOD2 (128³) [S2]. Radiation field sampling collects $N_s = 8$ uniform sampling points $x_p$ within each voxel $V_{i,j,k}$ [S2]. The voxel properties are calculated through the pre-trained implicit NeRF model $F_\theta$ with Fourier encoding $\gamma(\cdot)$: [S2]
$$ c_{i,j,k} = \frac{1}{N_s} \sum_{p=1}^{N_s} F_\theta(\gamma(x_p)) $$
$$ \sigma_{i,j,k} = \frac{1}{N_s} \sum_{p=1}^{N_s} F_\theta(\gamma(x_p)) $$

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Depends on the explicit storage structure, scaling with $D^3 \times F$ for voxel grids or the number of active entries for hash tables [S1].
*   **Time:** Feature retrieval operates in $O(1)$ time for both structures, while decoding complexity depends on the decoder architecture, such as $O(L \cdot F^2)$ for an MLP with $L$ layers [S1].

**Measured Performance:**
*   Hybrid representations address the computational expense of pure implicit NeRFs by enabling efficient reconstruction and rendering [S1].

**Strengths and Limitations:**
*   **Strengths:** Addresses the computational expense of pure implicit NeRFs [S1]. Efficient reconstruction and rendering [S1].
*   **Limitations:** Tri-linear interpolation may struggle with sharp discontinuities, requiring higher resolution or specialized handling [G].

**When to Use:**
*   When efficient radiance field reconstruction and rendering are required, balancing implicit compactness with explicit speed [S1].

## Background
Hybrid representations were developed to efficiently reconstruct and render radiance fields, addressing the slow training and rendering speed of pure implicit NeRFs [S1]. The framework leverages explicit features stored in structures like voxel grids, hash tables, or basis vectors, decoded by MLPs or spherical harmonics [S1].

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
[S2] Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework, https://www.informatica.si/index.php/informatica/article/download/12804/6562, fetched 2026-10-29
