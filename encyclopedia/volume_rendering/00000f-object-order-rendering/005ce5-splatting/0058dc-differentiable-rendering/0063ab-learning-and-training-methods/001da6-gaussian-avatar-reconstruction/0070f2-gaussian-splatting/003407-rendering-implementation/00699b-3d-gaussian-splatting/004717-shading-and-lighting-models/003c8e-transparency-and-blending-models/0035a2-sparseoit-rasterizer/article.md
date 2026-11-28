# SparseOIT Rasterizer

Also called: GPU software rasterization of 3D Gaussians with pre-rendering

## Overview
The SparseOIT Rasterizer is a rendering method for 3D Gaussian Splatting that computes a weighted, order-independent transparency (OIT) image and a pre-rendered image simultaneously [S1, unofficial]. It solves the problem of correct alpha blending in Gaussian splatting without requiring the expensive depth-sorting step inherent to standard rasterization [S1, unofficial]. Unlike standard 3DGS rasterization, which sorts Gaussians by distance to the camera to correctly accumulate transmittance, SparseOIT eliminates sorting entirely by using a blend-and-normalize (BAN) function and a blend-and-update (BAU) function, relying only on tile-ID ordering [S1, unofficial]. It builds on [Shading and Lighting Models](11924-shading-and-lighting-models.md).

## How it works
**Inputs:**
*   Image dimensions: width $w$, height $h$.
*   Gaussian attributes: 3D means $\mathbf{M}$, covariances $\mathbf{S}$, colors $\mathbf{C}$, opacities $\mathbf{A}$.
*   View matrix: $\mathbf{V}$.

**Outputs:**
*   Rendered image: $\mathbf{I}$.
*   Pre-rendered image state: $\mathbf{I}_{pre}$.

**Data Structures:**
*   $\mathbf{M}'$: Screen-space Gaussian means.
*   $\mathbf{S}'$: Screen-space Gaussian covariances.
*   $T$: List of tiles covering the image.
*   $L$: List of Gaussian data entries, duplicated and assigned tile keys.
*   $K$: List of tile keys corresponding to entries in $L$.
*   $R$: Ranges identifying which entries in $L$ belong to each tile.

**Parameters:**
*   **BAN (Blend and Normalize):** Function computing the final OIT color.
*   **BAU (Blend and Update):** Function updating the pre-rendered state for subsequent blending or accumulation.

**Algorithm Steps:**
1.  **Cull:** Remove Gaussians outside the view frustum using `CullGaussian(p, V)` [S1, unofficial].
2.  **Project:** Compute screen-space means $\mathbf{M}'$ and covariances $\mathbf{S}'$ from $\mathbf{M}$, $\mathbf{S}$, and $\mathbf{V}$ using `ScreenspaceGaussians` [S1, unofficial].
3.  **Tile:** Create a grid of tiles $T$ for the image dimensions $w, h$ using `CreateTiles` [S1, unofficial].
4.  **Assign Keys:** Duplicate Gaussian data into list $L$ and assign each entry a tile key based on its screen position, producing $K$, using `DuplicateWithKeys` [S1, unofficial].
5.  **Sort:** Sort entries in $L$ and $K$ solely by tile ID using `SortByKeys`. This replaces depth sorting [S1, unofficial].
6.  **Range:** Identify the start and end indices in $L$ for each tile using `IdentifyTileRanges`, producing $R$ [S1, unofficial].
7.  **Initialize:** Set the pre-rendered image $\mathbf{I}_{pre}$ to zero [S1, unofficial].
8.  **Rasterize:** Iterate over all tiles $t \in T$:
    9.  Iterate over all pixels $i$ in tile $t$:
        10. Retrieve the range $r$ of Gaussians affecting pixel $i$ using `GetTileRange` [S1, unofficial].
        11. Compute the final color $\mathbf{I}[i]$ using the BAN function:
            $$ \mathbf{I}[i] \leftarrow \text{BAN}(i, L, r, K, \mathbf{M}', \mathbf{S}', \mathbf{C}, \mathbf{A}, \mathbf{I}_{pre}[i]) $$
        12. Update the pre-rendered state using the BAU function:
            $$ \mathbf{I}_{pre}[i] \leftarrow \text{BAU}(i, L, r, K, \mathbf{M}', \mathbf{S}', \mathbf{C}, \mathbf{A}, \mathbf{I}_{pre}[i]) $$

**Formulas:**
The final color $\mathbf{C}$ is computed as [S1, unofficial]:
$$ \mathbf{C} = T \mathbf{c}_0 + (1-T)F $$
where:
*   $\mathbf{c}_0$ is the background color.
*   $T$ is the accumulated opacity/transmittance factor.
*   $F$ is the weighted average color:
    $$ F = \frac{P}{Q} $$
*   $P$ and $Q$ are sums over all $N$ Gaussians affecting the pixel:
    $$ P = \sum_{i=1}^{N} \mathbf{c}_i \alpha_i w_i $$
    $$ Q = \sum_{i=1}^{N} \alpha_i w_i $$
*   $\mathbf{c}_i$ is the color of the $i$-th Gaussian.
*   $\alpha_i$ is the opacity of the $i$-th Gaussian.
*   $w_i$ is the weight of the $i$-th Gaussian.

**Backward Propagation:**
The gradients for optimization are [S1, unofficial]:
$$ \frac{\partial \mathbf{C}}{\partial \mathbf{c}_i} = (1-T)\frac{\alpha_i w_i}{Q} $$
$$ \frac{\partial \mathbf{C}}{\partial \alpha_i} = \frac{T}{1-\alpha_i}(F-\mathbf{c}_0) + (1-T)\frac{w_i}{Q}(\mathbf{c}_i - F) $$
$$ \frac{\partial \mathbf{C}}{\partial w_i} = (1-T)\frac{\alpha_i}{Q}(\mathbf{c}_i - F) $$

**Pitfalls:**
*   The variables of each Gaussian are mutually independent in the backward propagation equation, which allows for parallel computation [S1, unofficial].
*   The update interval for the pre-rendering state may need to be increased (e.g., to 600) for specific scenes like "train", "room", and "bonsai" to ensure correct execution [S1, unofficial].

## Subtopics

### [Weighted Order-Independent Transparency for 3DGS](7772-weighted-order-independent-transparency-for-3dgs.md)
Weighted Order-Independent Transparency (OIT) for 3D Gaussian Splatting computes a blended image without sorting Gaussians by depth, using a learnable weight function to approximate correct transparency blending. It differs from standard 3DGS rasterization, which requires sorting by distance to camera to correctly accumulate transmittance, by eliminating the sorting step entirely in favor of a weighted average. The method builds on [SparseOIT Rasterizer](7562-sparseoit-rasterizer.md).

## Operating characteristics
Not given in the sources.

## Background
The SparseOIT Rasterizer was introduced in the paper "SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method" [S1, unofficial]. It addresses the limitations of standard order-dependent transparency in 3D Gaussian Splatting by providing an order-independent alternative that avoids the computational cost of sorting.

## Sources
[S1] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
