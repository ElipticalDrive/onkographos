# Mega-NeRF Spatial Partitioning

Also called: Cell-based NeRF, Spatial Data Parallelism

## Overview
Mega-NeRF Spatial Partitioning addresses the memory and computational bottlenecks of training Neural Radiance Fields (NeRF) on large-scale scenes by decomposing the environment into spatial cells, each managed by a separate MLP submodule trained in parallel [S1]. Unlike monolithic NeRF architectures, it isolates training data per cell based on ray intersection and visibility, enabling scalable construction of virtual fly-throughs for city-scale environments [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated input images with poses; a scene bounding volume.
*   **Outputs:** A set of MLP submodules $\{f^n\}$, one per spatial cell centroid $n \in \mathcal{N}$; appearance embeddings $\{l^{(a)}\}$ for each input image $a$.

**Data Structures:**
1.  **Centroids ($\mathcal{N}$):** A set of 3D points $n = (n_x, n_y, n_z)$ representing cell centers.
2.  **Submodules ($f^n$):** Independent MLPs. Each $f^n$ takes a 3D position $\mathbf{x}$ and outputs density $\sigma$. A second head (or the same network with additional inputs) takes $\mathbf{x}$, viewing direction $\mathbf{d}$, and appearance embedding $l^{(a)}$ to output color $\mathbf{c} = (r, g, b)$.
3.  **Appearance Embeddings ($l^{(a)}$):** A vector associated with each training image $a$ to model lighting variations.
4.  **Cell Trainsets:** A subset of training pixels/rays assigned to each cell.

**Algorithm Steps:**

1.  **Centroid Initialization:**
    Tessellate the scene into a top-down 2D grid to determine centroid locations. Fix the height ($z$-coordinate) of all centroids to a single value, as altitude variance is small relative to latitude/longitude differences in the target scenes [S1].

2.  **Submodule Initialization:**
    Initialize a set of MLP weights $f^n$ for each centroid $n \in \mathcal{N}$. Initialize appearance embeddings $l^{(a)}$ for each input image $a$ [S1].

3.  **Visibility Partitioning (Data Assignment):**
    For each training image $a$:
    a. Sample points along the camera rays corresponding to each pixel.
    b. Identify which spatial cells these sampled points intersect.
    c. Assign the pixel to the trainset of only those intersecting cells.
    d. Apply an overlap factor (e.g., 15%) between adjacent cells to minimize boundary artifacts [S1].
    
    *Note:* This step reduces the initial aggregate trainset size by approximately 10x per submodule [S1].

4.  **Foreground/Background Decomposition:**
    Subdivide the scene into a foreground volume (enclosing all camera poses) and a background volume (complementary area). Train separate Mega-NeRF instances for each volume, similar to NeRF++ [S1].

5.  **Spatial Data Pruning (Post-Coarse Training):**
    After an initial coarse training phase where the NeRF gains a rough understanding of scene geometry:
    a. Identify occluders (e.g., walls) inferred by specific cells.
    b. Prune pixels/rays from other cells if they are occluded by geometry in the current cell. For example, if a wall is inferred in cell $F$, prune pixels from cells $A$ and $B$ that are behind the wall [S1].
    
    *Note:* This further reduces trainset sizes by a factor of 2x [S1].

6.  **Parallel Training:**
    Train each submodule $f^n$ independently using its assigned trainset. The appearance embedding $l^{(a)}$ is shared across all cells for image $a$ but optimized jointly [S1].

**Query Time:**
Given a query point $\mathbf{x}$, direction $\mathbf{d}$, and image $a$:
1.  Find the closest centroid $n^*$:
    $$ n^* = \arg\min_{n \in \mathcal{N}} \| n - \mathbf{x} \|^2 $$
2.  Compute opacity and color using the corresponding submodule:
    $$ \sigma = f^{n^*}(\mathbf{x}) $$
    $$ \mathbf{c} = f^{n^*}(\mathbf{x}, \mathbf{d}, l^{(a)}) $$
    [S1]

**Parameters:**
*   **Overlap Factor:** 15% (used to minimize visual artifacts near cell boundaries) [S1].
*   **Centroid Height:** Fixed to a single value for the entire scene [S1].

**Edge Cases and Pitfalls:**
*   **Geometry Unknown at Initialization:** Initial pixel assignment is based on camera positions, not geometry, leading to irrelevant data in cells until coarse training occurs [S1].
*   **Altitude Variance:** The method assumes small altitude variance relative to horizontal spread. Scenes with significant vertical variation may require different centroid strategies (e.g., k-means or uncertainty-based partitioning, though these were explored and found less practical than the 2D grid) [S1].

## Operating characteristics
*   **Memory:** Scales linearly with the number of cells, as each cell stores its own MLP weights. This allows training on scenes that exceed the memory capacity of a single GPU [S1].
*   **Training Time:** Parallelizable across cells. Visibility partitioning reduces the data processed per cell by ~10x, and spatial pruning adds a further 2x reduction, significantly speeding up convergence per submodule [S1].
*   **Scalability:** Designed for large-scale, city-level scenes (e.g., North Pittsburgh) where standard NeRF fails due to memory constraints [S1].
*   **Limitations:** Requires a two-stage process (coarse training followed by pruning) to achieve optimal data reduction. The fixed-height centroid assumption limits applicability to scenes with significant vertical complexity [S1].

## Background
Mega-NeRF was introduced to enable the construction of large-scale NeRFs for virtual fly-throughs, addressing the scalability limits of the original NeRF [S1]. It builds on the NeRF++ architecture for foreground/background decomposition and incorporates appearance embeddings from NeRF in the Wild to handle lighting variations across large areas [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly-Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
