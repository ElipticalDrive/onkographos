# Mega-NeRF Spatial Partitioning

Also called: Cell-based NeRF, Spatial Data Parallelism

## Overview
Mega-NeRF Spatial Partitioning decomposes a large-scale 3D scene into a grid of spatial cells, each trained with a separate, lightweight Multi-Layer Perceptron (MLP) submodule in parallel [S1]. This approach solves the memory and training time bottlenecks of monolithic NeRFs by distributing representational capacity across the scene, enabling the construction of large-scale NeRFs for virtual fly-throughs [S1]. It sets itself apart from siblings like [KiloNeRF](8942-kilonerf.md) (which uses a single shared MLP with input-dependent gating) and [Sparse Radiance Field (SRF) Representation](8876-sparse-radiance-field-srf-representation.md) (which uses sparse voxel grids) by assigning distinct, independent MLP weights to fixed spatial regions, allowing for explicit spatial data parallelism and visibility-based data pruning [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated input images with poses; a scene bounding volume.
*   **Outputs:** A set of MLP submodules $\{f^n\}$, one per spatial cell centroid $n \in \mathcal{N}$; appearance embeddings $\{l^{(a)}\}$ for each image $a$.

**Data Structures:**
*   **Centroids ($\mathcal{N}$):** A set of 3D points $n = (n_x, n_y, n_z)$ representing the center of each spatial cell.
*   **Submodules ($f^n$):** Each $f^n$ is a sequence of fully connected layers (MLP) similar to the standard NeRF architecture [S1].
*   **Appearance Embeddings ($l^{(a)}$):** A vector associated with each input image $a$ to account for lighting variations [S1].
*   **Trainsets:** For each cell $n$, a subset of training pixels/rays assigned to that cell.

**Algorithm Steps:**

1.  **Centroid Selection:**
    *   Tessellate the scene into a top-down 2D grid [S1].
    *   Fix the height ($z$-coordinate) of all centroids to a single value, as altitude variance is small relative to latitude/longitude differences in the target scenes [S1].
    *   Define the set of centroids $\mathcal{N}$.

2.  **Initialization:**
    *   Initialize a corresponding MLP submodule $f^n$ for each centroid $n \in \mathcal{N}$ [S1].
    *   Initialize an appearance embedding vector $l^{(a)}$ for each input image $a$ [S1].

3.  **Visibility Partitioning (Data Assignment):**
    *   For each training image $a$, sample points along camera rays corresponding to each pixel [S1].
    *   Identify which spatial cells $n \in \mathcal{N}$ intersect with the ray segment within the scene bounds.
    *   Add the pixel to the trainset of only those intersecting cells [S1].
    *   Apply an overlap factor (e.g., 15%) between adjacent cells to minimize visual artifacts near boundaries [S1].

4.  **Spatial Data Pruning (Post-Coarse Training):**
    *   After an initial coarse training phase, use the inferred geometry to prune irrelevant pixels [S1].
    *   If a cell $n_{occluder}$ infers geometry (e.g., a wall) that occludes pixels assigned to other cells $n_{target}$, remove those pixels from $n_{target}$'s trainset [S1].

5.  **Rendering Query:**
    *   Given a query point $\mathbf{x}$, direction $\mathbf{d}$, and appearance embedding $l^{(a)}$:
    *   Find the closest centroid $\mathbf{n} = \arg\min_{n \in \mathcal{N}} \|\mathbf{n} - \mathbf{x}\|^2$ [S1].
    *   Compute opacity $\sigma$ and color $\mathbf{c}$ using the submodule associated with $\mathbf{n}$:
        $$
        \begin{aligned}
        \sigma &= f^{\mathbf{n}}(\mathbf{x}) \\
        \mathbf{c} &= f^{\mathbf{n}}(\mathbf{x}, \mathbf{d}, l^{(a)})
        \end{aligned}
        $$
    *   [S1]

**Parameters:**
*   **Overlap Factor:** 15% (used to minimize boundary artifacts) [S1].

**Edge Cases and Pitfalls:**
*   **Geometry Unknown at Initialization:** Initial pixel assignment is based on camera positions, not scene geometry, potentially assigning irrelevant pixels to cells [S1]. This is mitigated by the Spatial Data Pruning step [S1].
*   **Altitude Variance:** The method assumes small altitude variance relative to horizontal scale. If this assumption fails, fixing centroid height may lead to poor spatial partitioning [S1].

## Subtopics

### Scene Partitioning for Larger Scenes
Addresses model capacity limits when scaling Neural Irradiance Volumes (NIV) or similar representations to very large scenes by partitioning the environment into a grid of smaller regions and training a separate NIV per region [S2]. This distributes representational capacity to retain accuracy as scene size increases [S2]. It balances traditional spatial subdivision with neural network capacity allocation [S2]. A key pitfall is that scaling eventually hits model capacity limits, causing increased reconstruction error if not managed [S2]. Future directions include integrating level-of-detail techniques to allocate capacity adaptively or select lower-frequency models based on camera position [S2].

## Operating characteristics
*   **Time and Memory Complexity:** Reduces memory and training time for large-scale scenes by training separate MLP submodules in parallel [S1]. Visibility partitioning reduces the submodule trainset size by 10x compared to the initial aggregate trainset [S1]. Additional visibility pruning reduces trainset sizes by a further factor of 2x [S1].
*   **Measured Performance:** Demonstrated on large-scale scenes (e.g., North Pittsburgh) [S1].
*   **Strengths:** Enables scalable construction of large-scale NeRFs; reduces training data per submodule; handles lighting variations via appearance embeddings [S1].
*   **Limitations:** Model capacity can become a limiting factor for extremely large scenes if not partitioned finely enough [S2]; requires a two-stage process (coarse training then pruning) for optimal data reduction [S1].
*   **When to Use:** Large-scale scenes requiring virtual fly-throughs where monolithic NeRFs are too memory-intensive [S1].
*   **When Not to Use:** Small scenes where a single NeRF suffices; scenes with large altitude variance relative to horizontal scale [S1].

## Background
Mega-NeRF was introduced in 2022 by [S1]. It builds on [Specific Model Architectures](8585-specific-model-architectures.md) [G]. The method was developed to address the scalability issues of NeRF for large-scale environments, such as city-scale scenes, by decomposing the problem into spatially parallel sub-problems [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
[S2] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
