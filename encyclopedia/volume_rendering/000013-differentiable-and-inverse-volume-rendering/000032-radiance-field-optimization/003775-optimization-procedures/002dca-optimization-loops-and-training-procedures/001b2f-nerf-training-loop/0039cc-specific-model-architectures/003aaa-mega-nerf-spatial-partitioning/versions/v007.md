# Mega-NeRF Spatial Partitioning

Also called: Cell-based NeRF, Spatial Data Parallelism

## Overview
Mega-NeRF addresses the memory and computational bottlenecks of training Neural Radiance Fields on large-scale scenes by decomposing the scene into spatial cells, each managed by a separate MLP submodule trained in parallel [S1]. Unlike vanilla NeRF which uses a single global network, this method assigns query points to the nearest centroid, enabling scalable construction of virtual fly-throughs while maintaining visual fidelity through appearance embeddings and visibility-based data pruning [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses; a scene bounding volume.
*   **Outputs:** A set of MLP submodules $\{f^n\}$, one per spatial cell $n \in \mathcal{N}$, and their associated centroids $\mathcal{N}$.

**Data Structures:**
1.  **Centroids ($\mathcal{N}$):** A set of 3D points $\mathbf{n} = (n_x, n_y, n_z)$ representing the center of each spatial cell.
2.  **Submodules ($f^n$):** For each centroid $n$, a fully connected MLP similar to the standard NeRF architecture.
3.  **Appearance Embeddings ($l^{(a)}$):** A vector associated with each input image $a$ to model lighting variations.
4.  **Trainsets ($\mathcal{T}_n$):** For each cell $n$, a subset of training pixels/rays assigned to that cell.

**Algorithm Steps:**

1.  **Centroid Initialization:**
    Tessellate the scene into a top-down 2D grid to determine centroid positions. Fix the height ($z$-coordinate) of all centroids to a single value, as altitude variance is small relative to latitude/longitude differences in the target scenes [S1].

2.  **Submodule Initialization:**
    Initialize a separate MLP $f^n$ for each centroid $n \in \mathcal{N}$. Initialize appearance embeddings $l^{(a)}$ for each training image $a$ [S1].

3.  **Visibility Partitioning (Data Assignment):**
    For each training image $a$:
    a. Sample points along the camera rays corresponding to each pixel.
    b. Identify which spatial cells these rays intersect.
    c. Assign the pixel to the trainset $\mathcal{T}_n$ only for those intersecting cells $n$.
    d. Apply an overlap factor (e.g., 15%) between adjacent cells to minimize boundary artifacts [S1].

4.  **Spatial Data Pruning (Optional Post-Pruning):**
    After a coarse training phase where the scene geometry is partially understood:
    a. Identify occluders (e.g., walls) inferred by specific cells.
    b. Remove pixels/rays from cells that are occluded by geometry in other cells. For example, if cell $F$ contains a wall blocking the view from image 2 to cells $A$ and $B$, prune those pixels from $\mathcal{T}_A$ and $\mathcal{T}_B$ [S1].

5.  **Rendering and Optimization:**
    For a query ray with origin $\mathbf{o}$ and direction $\mathbf{d}$, sample points $\mathbf{x}_i$ along the ray. For each point $\mathbf{x}_i$:
    a. Find the nearest centroid $n^* = \arg\min_{n \in \mathcal{N}} \|\mathbf{n} - \mathbf{x}_i\|^2$.
    b. Query the corresponding submodule $f^{n^*}$ using the position $\mathbf{x}_i$, direction $\mathbf{d}$, and the appearance embedding $l^{(a)}$ of the current image:
       $$
       \sigma_i = f^{n^*}(\mathbf{x}_i)
       $$
       $$
       \mathbf{c}_i = f^{n^*}(\mathbf{x}_i, \mathbf{d}, l^{(a)})
       $$
    c. Compute the final color $\mathbf{C}(\mathbf{r})$ using the standard volume rendering integral [S1].
    d. Backpropagate gradients to update the weights of $f^{n^*}$ and the embedding $l^{(a)}$.

**Parameters:**
*   **Overlap Factor:** 15% (used to reduce visual artifacts at cell boundaries) [S1].
*   **Centroid Height:** Fixed to a single value for the scene [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Without the overlap factor, discontinuities may appear where rays cross from one cell's domain to another.
*   **Occlusion Errors:** Spatial data pruning relies on a "coarse understanding" of geometry. Pruning too early may remove valid data before the network learns the true scene structure [S1].
*   **Altitude Variance:** The method assumes small altitude variance relative to horizontal spread. Scenes with significant vertical variation may require a different centroid initialization strategy (e.g., 3D tessellation) [S1].

## Operating characteristics
*   **Memory Complexity:** Linear with respect to the number of cells and the size of each submodule. Since each submodule is smaller than a global NeRF, total memory usage is reduced, allowing for larger scenes [S1].
*   **Training Time:** Parallelizable across cells. Visibility partitioning reduces the effective training set size per cell by approximately 10x compared to the aggregate trainset. Additional spatial data pruning can further reduce it by 2x [S1].
*   **Strengths:** Scalable to large-scale outdoor scenes (e.g., city blocks); reduces memory footprint; handles lighting variations via appearance embeddings.
*   **Limitations:** Requires careful tuning of the overlap factor; centroid initialization strategy (2D grid) may not suit all scene geometries; pruning introduces dependency on intermediate training states [S1].

## Background
Mega-NeRF was introduced in 2022 by [S1] to enable the construction of large-scale NeRFs for virtual fly-throughs. It builds upon the foundational NeRF architecture [S1] and incorporates concepts from NeRF in the Wild regarding appearance embeddings [S1] and NeRF++ regarding foreground/background decomposition [S1]. The method was developed to overcome the limitations of single-network NeRFs when applied to scenes spanning large geographic areas.

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
