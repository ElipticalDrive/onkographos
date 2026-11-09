# Mega-NeRF Spatial Partitioning
Also called: Cell-based NeRF, Spatial Data Parallelism

## Overview
Mega-NeRF decomposes a large-scale 3D scene into disjoint spatial cells, each managed by an independent neural network submodule, to enable parallel training and inference for virtual fly-throughs [S1]. Unlike global NeRF models that suffer from memory bottlenecks and slow convergence on large scenes, this method localizes the prediction of opacity and color to the cell containing the query point, significantly reducing the training dataset size per module [S1]. It distinguishes itself from other sparse methods by using a fixed 2D tessellation of centroids rather than dynamic voxelization or hash grids, and by explicitly incorporating appearance embeddings to handle lighting variations across images [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated input images with known camera poses; a scene bounding volume.
*   **Outputs:** A set of neural network submodules $\{f^n\}$ associated with spatial centroids $\mathcal{N}$, and appearance embeddings $\{l^{(a)}\}$ for each image $a$.

**Data Structures:**
1.  **Centroids ($\mathcal{N}$):** A set of 3D points $\mathbf{n} = (n_x, n_y, n_z)$ representing the centers of spatial cells.
2.  **Submodules ($f^n$):** For each centroid $\mathbf{n}$, a fully connected neural network (MLP) similar to the standard NeRF architecture.
    *   Input: Position $\mathbf{x}$, direction $\mathbf{d}$, appearance embedding $l^{(a)}$.
    *   Output: Opacity $\sigma$ and color $\mathbf{c} = (r, g, b)$.
3.  **Appearance Embeddings ($l^{(a)}$):** A vector associated with each training image $a$ to model per-image lighting differences [S1].
4.  **Trainsets:** A mapping of pixels to the specific submodules (cells) they intersect.

**Algorithm Steps:**

1.  **Centroid Initialization:**
    Tessellate the scene into a top-down 2D grid to determine centroid locations. Fix the height ($z$-coordinate) of all centroids to a single value, assuming small altitude variance relative to latitude/longitude differences in the scene [S1].
    *   *Note:* The sources mention exploring k-means and uncertainty-based partitioning but select the 2D grid for simplicity and efficient query assignment [S1].

2.  **Submodule Initialization:**
    Initialize a separate MLP $f^n$ for each centroid $\mathbf{n} \in \mathcal{N}$. Initialize appearance embeddings $l^{(a)}$ for each input image $a$ [S1].

3.  **Visibility Partitioning (Data Assignment):**
    For each training image $a$:
    a. Sample points along the camera rays corresponding to each pixel.
    b. Identify which spatial cells (centroids) these rays intersect.
    c. Assign the pixel to the trainset of only those intersecting cells.
    d. Apply an overlap factor (typically 15%) between adjacent cells to minimize boundary artifacts [S1].
    *   *Result:* This reduces the trainset size per submodule by approximately 10x compared to the aggregate scene trainset [S1].

4.  **Coarse Training:**
    Train the submodules $f^n$ using the partitioned trainsets. During this phase, the model gains a coarse understanding of the scene geometry [S1].

5.  **Spatial Data Pruning (Visibility Pruning):**
    After coarse training, refine the trainsets based on inferred geometry:
    a. Identify occluders within each cell (e.g., a wall in cell F).
    b. Remove pixels from the trainsets of other cells (e.g., cells A and B) if they are occluded by geometry inferred by other cells.
    *   *Result:* This further reduces trainset sizes by a factor of 2x [S1].

6.  **Final Training:**
    Train the pruned submodules to convergence.

**Querying (Inference):**
Given a query point $\mathbf{x}$, viewing direction $\mathbf{d}$, and image appearance embedding $l^{(a)}$:
1.  Find the closest centroid $\mathbf{n}$:
    $$ \mathbf{n} = \argmin_{n \in \mathcal{N}} \lVert \mathbf{n} - \mathbf{x} \rVert^2 $$
2.  Retrieve opacity and color from the corresponding submodule $f^n$:
    $$ \sigma = f^n(\mathbf{x}) $$
    $$ \mathbf{c} = f^n(\mathbf{x}, \mathbf{d}, l^{(a)}) $$
3.  Composite the results using the standard volume rendering equation [S1].

**Parameters:**
*   **Overlap Factor:** 15% (used to minimize visual artifacts near cell boundaries) [S1].
*   **Centroid Height:** Fixed to a single value for the entire scene [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Without the overlap factor, discontinuities may appear at cell boundaries. The 15% overlap mitigates this [S1].
*   **Altitude Variance:** The method assumes small altitude variance. If the scene has significant vertical variation (e.g., multi-story buildings with distinct floors), fixing the centroid height may lead to poor spatial partitioning [S1].
*   **Occlusion Pruning:** Pruning relies on a "coarse understanding" of geometry. If the coarse training is insufficient, pruning may incorrectly remove valid training data [S1].

## Operating characteristics
*   **Memory:** Scales linearly with the number of cells and the size of individual MLPs, but significantly lower than a single global MLP for large scenes because each submodule only learns local features [S1].
*   **Training Time:** Parallelizable across cells. Visibility partitioning reduces the data volume per module by ~10x, and pruning adds another 2x reduction, accelerating convergence [S1].
*   **Inference:** Requires finding the nearest centroid, which can be optimized with spatial data structures (e.g., KD-trees or grids) given the fixed tessellation [S1].
*   **Strengths:** Scalable to large-scale scenes (e.g., city blocks); handles lighting variations via appearance embeddings; simple centroid selection [S1].
*   **Limitations:** Fixed 2D grid may not adapt well to scenes with complex vertical structures; requires two-stage training (coarse then pruned) [S1].

## Background
Mega-NeRF was introduced in 2022 by the authors of the paper "Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs" [S1]. It addresses the limitations of canonical NeRF and early sparse methods (like SRF and KiloNeRF) when applied to very large, outdoor scenes. The method builds on the concept of spatial data parallelism, allowing independent optimization of scene regions. It also incorporates techniques from NeRF++ for foreground/background decomposition and NeRF in the Wild for appearance embeddings [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
