# Mega-NeRF Spatial Partitioning

Also called: Cell-based NeRF, Spatial Data Parallelism

## Overview
Mega-NeRF Spatial Partitioning decomposes a large-scale scene into spatial cells, each managed by a separate, lightweight MLP submodule, to enable parallel training and reduce memory overhead [S1]. Unlike global NeRF models or single-voxel approaches, it assigns query points to the nearest centroid’s model, allowing independent optimization of scene regions [S1]. This method is specifically designed for large-scale virtual fly-throughs where global models become intractable [S1].

## How it works
**Inputs:** A set of calibrated training images with poses; a scene bounding volume.
**Outputs:** A set of MLP weight vectors $\{f^n\}$ for each cell $n \in \mathcal{N}$, and their corresponding centroid positions $\mathcal{N} = \{n_1, n_2, \dots\}$.

**Data Structures:**
1.  **Centroids ($\mathcal{N}$):** A list of 3D coordinates $(n_x, n_y, n_z)$. In the standard implementation, these are arranged in a top-down 2D grid with a fixed height $n_z$.
2.  **Submodule Weights ($f^n$):** For each centroid $n$, a set of weights representing a multi-layer perceptron (MLP). The architecture mirrors the original NeRF: fully connected layers with positional encoding inputs.
3.  **Appearance Embeddings ($l^{(a)}$):** A vector associated with each input image $a$, used to account for lighting variations.
4.  **Cell Trainsets ($T_n$):** A subset of training pixels/rays assigned to cell $n$.

**Algorithm Steps:**

1.  **Centroid Initialization:**
    Tessellate the scene into a top-down 2D grid. Fix the $z$-coordinate (height) of all centroids to a single value, justified by the small variance in altitude relative to latitude/longitude differences in the target scenes [S1]. Let $\mathcal{N}$ be the set of resulting centroids.

2.  **Model Initialization:**
    Initialize a distinct MLP submodule $f^n$ for each centroid $n \in \mathcal{N}$. Initialize appearance embeddings $l^{(a)}$ for each training image $a$ [S1].

3.  **Visibility Partitioning (Data Assignment):**
    For each training image $a$:
    a. Sample camera rays through the image.
    b. For each ray, determine which spatial cells it intersects.
    c. Assign the ray/pixel to the trainset $T_n$ only for those intersecting cells $n$.
    d. Apply an overlap factor (e.g., 15%) between adjacent cells to minimize boundary artifacts [S1].
    
    *Note:* This step reduces the initial aggregate trainset size by approximately 10x per submodule [S1].

4.  **Spatial Data Pruning (Optional, Post-Coarse Training):**
    After an initial coarse training phase where the NeRFs gain a rough understanding of geometry:
    a. Identify occluders (e.g., walls) inferred by specific cells.
    b. Prune pixels/rays from other cells that are occluded by these inferred geometries.
    c. This further reduces trainset sizes by an additional factor of 2x [S1].

5.  **Parallel Optimization:**
    Train each submodule $f^n$ independently using its assigned trainset $T_n$ and the appearance embeddings $l^{(a)}$ of the source images. The loss is computed per pixel based on the predicted color and opacity.

**Query Function:**
At inference time, for a query point $\mathbf{x}$, direction $\mathbf{d}$, and image $a$:
1.  Find the closest centroid:
    $$ \mathbf{n} = \arg\min_{n \in \mathcal{N}} \|\mathbf{n} - \mathbf{x}\|^2 $$
2.  Compute opacity $\sigma$ and color $\mathbf{c}$ using the weights of the closest submodule:
    $$ \sigma = f^{\mathbf{n}}(\mathbf{x}) $$
    $$ \mathbf{c} = f^{\mathbf{n}}(\mathbf{x}, \mathbf{d}, l^{(a)}) $$
    where $\mathbf{n}$ is the index of the closest centroid [S1].

**Parameters:**
*   **Overlap Factor:** 15% (used to reduce visual artifacts at cell boundaries) [S1].
*   **Centroid Height:** Fixed to a single value for all cells in the grid [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Without the overlap factor, discontinuities may appear between cells. [S1]
*   **Occlusion Pruning:** Pruning relies on coarse geometry estimates. If the coarse model is inaccurate, valid rays may be incorrectly pruned. [S1]
*   **Altitude Variance:** The fixed-height centroid assumption works only when altitude variance is small relative to horizontal scale. [S1]

## Operating characteristics
*   **Time Complexity:** Training is parallelized across cells. The total training time is dominated by the slowest cell, but overall efficiency improves due to reduced data per cell (10x reduction initially, up to 20x with pruning) [S1].
*   **Memory Complexity:** Memory usage scales with the number of cells and the size of each local MLP, rather than a single global model. This allows training on scenes that exceed the memory capacity of a single global NeRF [S1].
*   **Strengths:** Scalability to large-scale scenes (e.g., city blocks); parallelizable training; reduced memory footprint per GPU [S1].
*   **Limitations:** Potential visual artifacts at cell boundaries if overlap is insufficient; dependency on coarse geometry for effective pruning; fixed-height assumption limits applicability to scenes with significant vertical variation [S1].
*   **When to use:** Large-scale outdoor scenes requiring virtual fly-throughs where global NeRF training is infeasible [S1].
*   **When not to use:** Small-scale scenes where global NeRF is efficient; scenes with large vertical extent relative to horizontal scale (due to fixed-height centroid assumption) [S1].

## Background
Mega-NeRF was introduced in 2022 by [S1] to address the scalability limits of NeRF for large-scale environments. It builds on the principle of spatial data parallelism, decomposing the scene to allow independent training of local models. The method draws inspiration from NeRF++ for foreground/background decomposition and uses a grid-based centroid selection for simplicity and efficiency [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
