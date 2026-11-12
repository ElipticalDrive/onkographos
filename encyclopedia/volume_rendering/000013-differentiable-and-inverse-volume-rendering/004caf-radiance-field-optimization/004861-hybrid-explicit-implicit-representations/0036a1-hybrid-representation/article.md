# Hybrid Representation

Also called: Explicit-Implicit Hybrid

## Overview
Hybrid Representation is a framework for radiance field optimization that combines explicit feature storage with an implicit decoder to efficiently reconstruct and render scenes, addressing the computational expense of pure implicit NeRFs [S1]. It sets itself apart from siblings like DVGO by providing a general architectural pattern where spatial features are stored in a discrete structure (e.g., voxel grid, hash table) and decoded via a neural network, rather than relying solely on dense voxel grids or purely implicit MLPs [S1].

## How it works
**Inputs:**
*   A set of training images with known camera poses.
*   A scene bounding box defining the spatial domain.

**Outputs:**
*   A trained Explicit Feature Storage $V$.
*   A trained Decoder $D$.
*   A rendered image for a given query ray.

**Data Structures:**
1.  **Explicit Feature Storage ($V$):**
    *   **Voxel Grid:** A 3D array of dimensions $D_x \times D_y \times D_z$. Each cell stores a feature vector of dimension $F$. [S1]
    *   **Hash Table:** A mapping from spatial coordinates to feature vectors. Requires a hash function $H$ and a capacity $C$. [S1]
    *   **Basis Vectors:** A set of learned basis functions/matrices. [S1]
2.  **Decoder ($D$):**
    *   Typically a Multi-Layer Perceptron (MLP) or Spherical Harmonics (SH) coefficients. [S1]

**Algorithm Steps:**
1.  **Initialize Storage:** Construct $V$. If using a voxel grid, initialize dimensions and feature dimensions. If using a hash table, initialize the hash function and capacity. [S1]
2.  **Feature Retrieval:** For a query point $p = (x, y, z)$ in 3D space, retrieve the feature vector $v_p$ from $V$.
    *   If $V$ is a voxel grid, perform tri-linear interpolation of the surrounding voxel features to obtain $v_p$ [S1].
    *   If $V$ is a hash table, compute the hash of $p$ and retrieve the associated feature vector. [S1]
3.  **Decoding:** Pass $v_p$ through the decoder $D$ to predict volume density $\sigma(p)$ and color $c(p, d)$, where $d$ is the viewing direction [S1].
    *   $\sigma(p) = D_\sigma(v_p)$
    *   $c(p, d) = D_c(v_p, d)$
    *   The decoder $D$ can be an MLP [S1] or Spherical Harmonics [S1].
4.  **Rendering:** Use standard volume rendering integration along rays passing through the scene, summing the contributions of $\sigma$ and $c$ weighted by transmittance. [S1]

**Parameters:**
*   **Storage Type:** Voxel grid, hash table, or basis vectors [S1].
*   **Decoder Type:** MLP or Spherical Harmonics [S1].
*   **Interpolation Method:** Tri-linear interpolation is standard for voxel grids [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Tri-linear interpolation assumes continuous variation between voxels; sharp discontinuities may require higher resolution or specialized handling [S1].
*   **Boundaries:** Points outside the defined storage domain must be handled (e.g., clamping, zero-padding, or out-of-bounds flags) [G].
*   **Degenerate Input:** Empty regions in the scene may result in sparse feature storage, requiring efficient sparse data structures or padding strategies [S1].

## Subtopics

### HybridNeRF
HybridNeRF is a radiance field optimization method that combines explicit surface representations with implicit volumetric modeling to achieve high-fidelity novel-view synthesis at real-time frame rates [S1]. It addresses the computational expense of pure implicit NeRFs and the reconstruction limitations of pure surface-based methods by rendering most of the scene as thin surfaces while modeling challenging regions (thin structures, semi-opaque areas) volumetrically using spatially-varying surfaceness parameters $\beta(\mathbf{x})$ [S1]. Unlike prior work that treats surfaceness as a global parameter, HybridNeRF learns $\beta$ in a spatially adaptive manner, allowing it to improve error rates by 15–30% compared to state-of-the-art baselines while achieving at least 36 FPS for 2K×2K virtual-reality resolutions [S1].

### Baking Implicit NeRF into Multi-resolution Voxel Mesh
A technique to replace the computational bottleneck of MLP inference in implicit NeRF by pre-calculating voxel properties (color and density) into a multi-resolution voxel grid structure [S2]. Spatial division divides scene space into a three-level voxel grid based on adaptive resolution. The voxel side length is defined as $s_k = \frac{S}{2^k}$ where $S$ is the maximum side length of the scene and $k=0, 1, 2$ correspond to LOD0 (512³), LOD1 (256³), and LOD2 (128³) [S2]. Radiation field sampling collects $N_s = 8$ uniform sampling points $x_p$ within each voxel $V_{i,j,k}$. The voxel properties are calculated through the pre-trained implicit NeRF model $F_\theta$ with Fourier encoding $\gamma(\cdot)$:
$$ c_{i,j,k} = \frac{1}{N_s} \sum_{p=1}^{N_s} F_\theta(\gamma(x_p)) $$
$$ \sigma_{i,j,k} = \frac{1}{N_s} \sum_{p=1}^{N_s} F_\theta(\gamma(x_p)) $$
[S2]

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** Depends on the explicit storage. Voxel grids scale with $D^3 \times F$. Hash tables scale with the number of active entries. [S1]
*   **Time:** Feature retrieval is $O(1)$ for hash tables and $O(1)$ for voxel grids (with interpolation). Decoding is $O(L \cdot F^2)$ for an MLP with $L$ layers. [S1]

**Measured Performance:**
*   HybridNeRF achieves at least 36 FPS for 2K×2K virtual-reality resolutions [S1].

**Strengths and Limitations:**
*   **Strengths:** Addresses the computational expense of pure implicit NeRFs [S1]. Efficient reconstruction and rendering [S1].
*   **Limitations:** Tri-linear interpolation may struggle with sharp discontinuities [S1].

**When to Use:**
*   When efficient reconstruction and rendering of radiance fields are required, balancing the compactness of implicit representations with the speed of explicit structures [S1].

## Background
Hybrid representations were developed to efficiently reconstruct and render radiance fields, addressing the slow training and rendering speed of pure implicit NeRFs [S1]. The framework leverages explicit features stored in structures like voxel grids, hash tables, or basis vectors, decoded by MLPs or spherical harmonics [S1].

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
[S2] Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework, https://www.informatica.si/index.php/informatica/article/download/12804/6562, fetched 2026-10-29
