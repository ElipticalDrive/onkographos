# Mega-NeRF Spatial Partitioning

Also called: Cell-based NeRF, Spatial Data Parallelism

## Overview
Mega-NeRF Spatial Partitioning decomposes a large-scale scene into spatial cells, each managed by a separate, lightweight MLP submodule, to enable parallel training and reduce memory consumption [S1]. Unlike monolithic NeRF models or methods that rely on global feature grids, it assigns each query point to the single closest centroid, allowing for independent optimization of local geometry and appearance [S1]. This approach specifically addresses the scalability limits of standard NeRF when applied to city-scale environments [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated training images with poses; a scene bounding volume.
*   **Outputs:** A set of centroids $\mathcal{N}$, a set of MLP weight submodules $\{f^n\}_{n \in \mathcal{N}}$, and appearance embeddings $\{l^{(a)}\}$ for each image $a$.

**Data Structures:**
*   **Centroids:** A set $\mathcal{N} = \{n_1, n_2, \dots, n_N\}$ where each $n \in \mathbb{R}^3$ represents the center of a spatial cell.
*   **Submodules:** Each $f^n$ is a Multi-Layer Perceptron (MLP) with fully connected layers, identical in architecture to the base NeRF network but with significantly fewer parameters due to local specialization [S1].
*   **Appearance Embeddings:** A vector $l^{(a)} \in \mathbb{R}^k$ associated with each training image $a$ to capture lighting variations [S1].
*   **Trainsets:** For each submodule $f^n$, a subset of training pixels is assigned based on spatial intersection [S1].

**Algorithm Steps:**

1.  **Centroid Initialization:**
    Tessellate the scene into a top-down 2D grid to determine centroid positions [S1]. Due to small altitude variance relative to latitude/longitude differences in typical aerial/drone captures, fix the height ($z$-coordinate) of all centroids to a single value [S1]. Let the set of centroids be $\mathcal{N}$ [S1].

2.  **Submodule Initialization:**
    Initialize a separate MLP $f^n$ for each centroid $n \in \mathcal{N}$ [S1]. Initialize appearance embeddings $l^{(a)}$ for each training image $a$ [S1].

3.  **Visibility Partitioning (Data Assignment):**
    For each training image $a$:
    a. Sample points along the camera rays corresponding to each pixel [S1].
    b. Identify which spatial cells (centroids) these rays intersect [S1].
    c. Assign the pixel to the trainset of only those intersecting cells [S1].
    d. Apply an overlap factor (typically 15%) between adjacent cells to minimize boundary artifacts [S1].
    *Note:* This step reduces the initial aggregate trainset size by approximately 10x per submodule [S1].

4.  **Coarse Training:**
    Train all submodules $f^n$ in parallel using their assigned local trainsets [S1]. This provides a coarse understanding of the scene geometry [S1].

5.  **Spatial Data Pruning (Visibility Refinement):**
    After coarse training, prune pixels from submodules if they are occluded by geometry inferred by other cells [S1]. For example, if a wall is inferred in cell $F$, pixels from other cells ($A, B$) that are behind this wall are removed from their respective trainsets [S1]. This further reduces trainset sizes by a factor of 2x [S1].

6.  **Fine Training:**
    Continue training the submodules in parallel using the pruned trainsets until convergence [S1].

**Querying:**
At inference time, for a query point $\mathbf{x}$, direction $\mathbf{d}$, and image $a$:
1.  Find the closest centroid $n^*$:
    $$ n^* = \arg\min_{n \in \mathcal{N}} \| n - \mathbf{x} \|^2 $$
2.  Compute opacity $\sigma$ and color $\mathbf{c}$ using the corresponding submodule $f^{n^*}$:
    $$ \sigma = f^{n^*}(\mathbf{x}) $$
    $$ \mathbf{c} = f^{n^*}(\mathbf{x}, \mathbf{d}, l^{(a)}) $$
    [S1]

**Parameters:**
*   **Overlap Factor:** 15% (used to minimize visual artifacts near cell boundaries) [S1].
*   **Centroid Height:** Fixed to a single value for the scene [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion Handling:** Initial pixel assignment is based on camera positions, not geometry. Pruning is required after coarse training to avoid training submodules on occluded data [S1].
*   **Altitude Variance:** The method assumes small altitude variance. If significant vertical variation exists, the fixed-height centroid assumption may degrade performance [S1].
*   **Boundary Artifacts:** Without the overlap factor, discontinuities may appear at cell boundaries [S1].

## Operating characteristics
*   **Time Complexity:** Training time is reduced due to parallelization of submodules and smaller local trainsets. Visibility partitioning reduces trainset size by 10x, and pruning adds a further 2x reduction [S1].
*   **Memory Complexity:** Memory usage is significantly lower than monolithic NeRF because each submodule is small and only stores weights for its local region [S1].
*   **Scalability:** Designed for large-scale scenes (e.g., city-scale) where standard NeRF fails due to memory constraints [S1].
*   **Strengths:** Parallelizable training, reduced memory footprint, handles large scenes efficiently [S1].
*   **Limitations:** Requires careful handling of cell boundaries; assumes relatively flat terrain or fixed altitude for centroid placement [S1].

## Background
Mega-NeRF was introduced to address the scalability issues of NeRF in large-scale virtual fly-throughs [S1]. It builds on the concept of spatial partitioning and appearance embeddings seen in NeRF in the Wild [S1]. The method was developed to enable the construction of NeRFs for entire cities, such as North Pittsburgh [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
