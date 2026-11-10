# Hybrid Representation

Also called: Explicit-Implicit Hybrid

## Overview
Hybrid Representation is a framework for radiance field optimization that combines explicit feature storage with an implicit decoder to efficiently reconstruct and render scenes [S1]. It addresses the computational expense of pure implicit Neural Radiance Fields (NeRF) by storing features in a structured format (e.g., voxel grid, hash table) and decoding them via a lightweight network [S1]. Unlike pure implicit methods that rely solely on MLP inference for every point, or pure explicit methods that lack the flexibility of continuous representation, hybrid methods balance memory efficiency and rendering speed by leveraging cheap lookups for feature retrieval [S1].

## How it works
**Inputs:** A set of training images with known camera poses; an initial explicit feature storage structure $V$.
**Outputs:** A radiance field model capable of rendering novel views by computing density $\sigma$ and color $c$ for any 3D point.

**Data Structures:**
1.  **Explicit Feature Storage ($V$):** A data structure holding feature vectors. Common implementations include:
    *   Voxel Grid: A 3D array where each cell stores a feature vector.
    *   Hash Table: A mapping from spatial coordinates to feature vectors.
    *   Basis Vectors/Matrices: A set of learned basis functions.
2.  **Decoder ($D$):** A function (typically a Multi-Layer Perceptron (MLP) or Spherical Harmonics) that maps the retrieved feature vector $v_p$ to physical properties.

**Algorithm Steps:**
1.  **Initialize Storage:** Construct the explicit feature storage $V$. If using a voxel grid, initialize the grid dimensions and feature dimensions. If using a hash table, initialize the hash function and capacity.
2.  **Feature Retrieval:** For a query point $p = (x, y, z)$ in 3D space, retrieve the corresponding feature vector $v_p$ from $V$.
    *   If $V$ is a voxel grid, perform tri-linear interpolation of the surrounding voxel features to obtain $v_p$ [S1].
    *   If $V$ is a hash table, compute the hash of $p$ and retrieve the associated feature vector.
3.  **Decoding:** Pass the feature vector $v_p$ through the decoder $D$ to predict the volume density $\sigma(p)$ and color $c(p, d)$, where $d$ is the viewing direction.
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

### HybridNeRF
HybridNeRF is a radiance field optimization method that combines explicit surface representations with implicit volumetric modeling to achieve high-fidelity novel-view synthesis at real-time frame rates [S1]. It addresses the computational expense of pure implicit NeRFs and the reconstruction limitations of pure surface-based methods by rendering most of the scene as thin surfaces while modeling challenging regions (thin structures, semi-opaque areas) volumetrically using spatially-varying surfaceness parameters $\beta(\mathbf{x})$ [S1]. Unlike prior work that treats surfaceness as a global parameter, HybridNeRF learns $\beta$ in a spatially adaptive manner, allowing it to improve error rates by 15–30% compared to state-of-the-art baselines while achieving at least 36 FPS for 2K×2K virtual-reality resolutions [S1].

### Baking Implicit NeRF into Multi-resolution Voxel Mesh
A technique to replace the computational bottleneck of MLP inference in implicit NeRF by pre-calculating voxel properties (color and density) into a multi-resolution voxel grid structure [S2].
**Steps:**
1.  **Spatial Division:** Divide the scene space into a three-level voxel grid based on adaptive resolution. Define voxel side length $s_k = \frac{S}{2^k}$ where $S$ is the maximum side length of the scene and $k=0, 1, 2$ correspond to LOD0 (512³), LOD1 (256³), and LOD2 (128³) [S2]. The size of the voxels is positively correlated with the local detail density of the scene [S2].
2.  **Radiation Field Sampling:** For each voxel $V_{i,j,k}$, collect $N_s = 8$ sampling points $x_p$ ($p = 1, 2, \dots, 8$) uniformly within it [S2].
3.  **Property Calculation:** Calculate voxel properties (color $c$, density $\sigma$) through the pre-trained implicit NeRF model $F_\theta$ with positional encoding $\gamma(\cdot)$:
    $$ c_{i,j,k} = \frac{1}{N_s} \sum_{p=1}^{N_s} F_\theta(\gamma(x_p)) \} $$
    $$ \sigma_{i,j,k} = \frac{1}{N_s} \sum_{p=1}^{N_s} F_\theta(\gamma(x_p)) \} $$
    [S2]

## Operating characteristics
*   **Time Complexity:** Rendering speed is significantly faster than pure implicit NeRFs because feature retrieval is $O(1)$ or $O(\log N)$ depending on the storage structure, avoiding expensive MLP evaluations for every sample point [S1]. Training speed depends on the decoder complexity and the frequency of feature updates [S1].
*   **Memory Complexity:** Memory usage is determined by the size of the explicit storage $V$. Voxel grids consume fixed memory proportional to resolution, while hash tables are more memory-efficient for sparse scenes [S1].
*   **Strengths:** High rendering efficiency, ability to handle large scenes, and flexibility in combining explicit and implicit advantages [S1].
*   **Limitations:** Requires careful tuning of storage resolution and decoder capacity; may suffer from aliasing or blurring if the explicit storage is too coarse [S1].
*   **When to Use:** Suitable for applications requiring real-time rendering or efficient large-scale scene reconstruction where pure implicit NeRFs are too slow [S1].
*   **When Not to Use:** Not ideal for scenes with extremely fine, high-frequency details that cannot be captured by the chosen explicit storage resolution without prohibitive memory costs [S1].

## Background
Hybrid representations were developed to address the computational expense of pure implicit NeRFs, which are highly compact but slow to train and render [S1]. The framework leverages efficient free-viewpoint video rendering techniques and builds on the rendering scheme proposed by NeRF [S1]. Early implementations used voxel grids, hash tables, or basis vectors to store features, with decoders ranging from MLPs to spherical harmonics [S1]. The approach has been widely adopted in subsequent works for static and dynamic scene reconstruction [S1].

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
[S2] Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework, https://www.informatica.si/index.php/informatica/article/download/12804/6562, fetched 2026-10-29
