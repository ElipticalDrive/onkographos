# MegaNeRF

Also called: Altitude-constrained Raycasting, Ellipsoid Partitioning, Mega-NeRF, Mega-NeRF Foreground/Background Decomposition

## Overview
MegaNeRF is a framework for training large-scale 3D scenes (spanning $150,000$ to $1,300,000 \text{ m}^2$) that supports interactive human-in-the-loop fly-throughs from drone footage [S1]. It addresses the infeasibility of training a single monolithic NeRF on thousands of images with varying lighting and large capacities by decomposing the scene into spatial cells, each modeled by an independent NeRF submodule [S1]. Unlike siblings such as [BlockNeRF](8581-blocknerf.md) and [KiloNeRF](8581-blocknerf.md) which use spatial partitioning primarily to speed up inference of a pre-trained network, MegaNeRF uses partitioning to enable data parallelism for scalable training, as obtaining a pre-trained NeRF for such large scales is infeasible [S1]. It further improves upon [NeRF++](8581-blocknerf.md) by modeling the foreground with a tight ellipsoid enclosing camera poses and using camera altitude constraints to skip underground sampling [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $N$ training images with known camera poses (position $\mathbf{t}$ and orientation $\mathbf{R}$) and intrinsics. For altitude-constrained rendering, camera altitude measurements $h$ are also required [S1].
- **Outputs:** A set of $K$ independent NeRF submodules $\{\mathcal{M}_k\}_{k=1}^K$, where each $\mathcal{M}_k$ consists of an MLP predicting density $\sigma$ and color $\mathbf{c}$ for points within its assigned spatial cell $V_k$ [S1].

**Data Structures:**
1.  **Spatial Partitioning:** The scene volume $\mathcal{V}$ is partitioned into $K$ non-overlapping spatial cells $\{V_k\}_{k=1}^K$. MegaNeRF employs two levels of partitioning:
    -   **Foreground/Background Split:** The scene is divided into a foreground volume enclosing all camera poses and a background volume for the complement [S1].
    -   **Cell Partitioning:** The foreground volume is further subdivided via spatial Voronoi partitioning into cells [S1].
2.  **Submodule Assignment:** Each cell $V_k$ is associated with a dedicated MLP $\mathcal{M}_k$ [S1].
3.  **Pixel Partitioning:** A mapping from training pixels to cells. For each training image $i$, a set of pixels $P_i$ is assigned to the subset of cells $\{V_k\}$ that the ray corresponding to that pixel intersects [S1].

**Algorithm Steps:**

1.  **Scene Decomposition and Boundary Definition:**
    -   Define the foreground volume $V_{fg}$ as an ellipsoid that tightly encloses all camera poses and relevant foreground detail [S1]. This replaces the unit sphere used in NeRF++ [S1].
    -   Define the background volume $V_{bg}$ as the complement of $V_{fg}$ [S1].
    -   Partition $V_{fg}$ into $K$ cells $\{V_k\}$ using spatial Voronoi partitioning [S1]. The centroids of these cells are fixed at the same height if the altitude variation between camera poses is small relative to latitude/longitude differences [S1].

2.  **Geometric Data Partitioning (Training Data Preparation):**
    -   For each training image $i$ with camera pose $C_i$:
        -   Cast rays through each pixel $p \in P_i$.
        -   Determine the set of cells $\mathcal{K}_p = \{k \mid \text{ray}(p, C_i) \cap V_k \neq \emptyset\}$ [S1].
        -   Assign pixel $p$ to the training set of every submodule $\mathcal{M}_k$ where $k \in \mathcal{K}_p$ [S1].
    -   This visibility-based partitioning ensures that each submodule is trained only on data relevant to its spatial region, reducing the size of each submodule's trainset by approximately 10x [S1].

3.  **Parallel Submodule Training:**
    -   Train each submodule $\mathcal{M}_k$ independently and in parallel using its assigned partitioned pixel data [S1].
    -   The loss function for each submodule is the standard NeRF photometric loss computed over its assigned pixels [S1].

4.  **Inference (Raycasting and Rendering):**
    -   For a query ray $\mathbf{r}$ from a novel viewpoint:
        -   Intersect $\mathbf{r}$ with the foreground ellipsoid $V_{fg}$ and background volume $V_{bg}$ [S1].
        -   **Foreground Sampling:**
            -   If altitude constraints are enabled, determine the ground level $h_{ground}$ for the ray's intersection with the terrain. Terminate sampling if the ray depth exceeds the altitude constraint (avoiding underground regions) [S1].
            -   Sample points $\mathbf{x}_j$ along the ray segment within $V_{fg}$.
            -   Identify the cell $V_k$ containing $\mathbf{x}_j$.
            -   Query the corresponding submodule $\mathcal{M}_k$ to obtain $\sigma_j = \sigma_k(\mathbf{x}_j)$ and $\mathbf{c}_j = \mathbf{c}_k(\mathbf{x}_j)$ [S1].
        -   **Background Sampling:**
            -   Sample points $\mathbf{x}_m$ along the ray segment within $V_{bg}$ using the 4D outer volume parameterization (inverse sphere mapping) as defined in NeRF++ [S1].
            -   Query the background submodule $\mathcal{M}_{bg}$ to obtain $\sigma_m$ and $\mathbf{c}_m$ [S1].
        -   **Volume Rendering:**
            -   Combine foreground and background samples sorted by distance from the ray origin.
            -   Compute the final color $C(\mathbf{r})$ using the standard volume rendering integral:
                $$
                C(\mathbf{r}) = \sum_{i} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) \mathbf{c}_i
                $$
                where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance [S1, S3].

5.  **Interactive Rendering Optimization (Temporal Coherence):**
    -   To achieve near-interactive view generation, exploit temporal coherence by caching occupancy and color values from nearby previous views [S1].
    -   This "just-in-time" visualization technique reduces the need for full raycasting and sampling for every frame, accelerating rendering by up to 40x compared to conventional NeRF while remaining within 0.8 dB in PSNR quality [S1].

**Parameters:**
-   **Number of Cells ($K$):** Not given in the sources.
-   **Ellipsoid Parameters:** Defined by the bounding box of camera poses [S1].
-   **Altitude Threshold:** Determined by camera altitude measurements $h$ [S1].

**Edge Cases and Pitfalls:**
-   **Underground Sampling:** Without altitude constraints, rays may query undefined underground regions, leading to artifacts. MegaNeRF mitigates this by terminating rays near ground level [S1].
-   **Boundary Discontinuities:** As with all partitioned methods, discontinuities may arise at cell boundaries. The sources do not specify a specific boundary smoothing technique for MegaNeRF, only that it uses standard raycasting and querying the encompassing submodule [S1]. [G] In practice, this may require blending or overlap regions, but this is not detailed in the provided sources.

## Operating characteristics
-   **Training Speed:** Improves training speed by 3x compared to baseline approaches (monolithic NeRF) on existing datasets [S1].
-   **Rendering Speed:** Achieves a 40x speedup over conventional NeRF rendering when combined with the temporal coherence caching technique [S1].
-   **Quality:** Remains within 0.8 dB in PSNR quality compared to conventional NeRF [S1]. Improves PSNR by 12% compared to baseline approaches on existing datasets [S1].
-   **Scalability:** Scales to scenes 1000x larger than prior work, handling 150,000 to over 1,300,000 $\text{m}^2$ per scene [S1].
-   **Strengths:** Enables data parallelism for scalable training of large scenes; tight ellipsoid bounds and altitude constraints improve sampling efficiency; temporal coherence enables interactive fly-throughs.
-   **Limitations:** Requires geometric clustering and visibility analysis for data partitioning; relies on accurate camera altitude measurements for optimal performance; potential for visual artifacts at cell boundaries if not handled (not specified in sources).

## Background
MegaNeRF was introduced in 2022 by [S1]. It was motivated by the need for search-and-rescue applications where drones provide inexpensive means to quickly survey large areas. The method builds on [NeRF++](8581-blocknerf.md) by replacing the unit sphere foreground with an ellipsoid and adding altitude constraints. It differs from [DeRF](8581-blocknerf.md) and [KiloNeRF](8581-blocknerf.md) by using spatial partitioning for data parallelism during training rather than for inference acceleration of a pre-trained network [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly- Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S3] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
