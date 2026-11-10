# MegaNeRF

Also called: Altitude-constrained Raycasting, Ellipsoid Partitioning, Mega-NeRF, Mega-NeRF Foreground/Background Decomposition

## Overview
MegaNeRF is a framework for training large-scale 3D scenes that support interactive human-in-the-loop fly-throughs, addressing challenges of modeling thousands of images with varying lighting, large model capacities, and fast rendering [S1]. It differs from sibling methods like BlockNeRF and KiloNeRF, which use spatial partitioning primarily to speed up inference of a pre-trained single network, by using spatial partitioning to enable data parallelism for scalable training, as obtaining a pre-trained NeRF for large scales is infeasible [S1]. It also improves upon NeRF++ by using an ellipsoid for tighter foreground bounds and adding altitude constraints to skip underground sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses and intrinsics.
*   **Outputs:** $K$ independent NeRF submodules $\{\mathcal{M}_k\}_{k=1}^K$, where each $\mathcal{M}_k$ consists of a density network $\sigma_k$ and a color network $\mathbf{c}_k$.

**Data Structures:**
1.  **Spatial Cells:** The scene volume is partitioned into $K$ non-overlapping spatial cells $\{V_k\}_{k=1}^K$. In the standard implementation, this is done via spatial Voronoi partitioning [S1].
2.  **Submodules:** Each cell $V_k$ has an associated MLP $\mathcal{M}_k$. The MLP takes a 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as input and outputs density $\sigma$ and color $\mathbf{c}$ [S1].
3.  **Foreground/Background Decomposition:** The scene is further subdivided into a foreground volume enclosing all camera poses and a background volume for the complement [S1].
    *   **Foreground:** Modeled as an ellipsoid that tightly encloses camera poses and relevant detail, replacing NeRF++'s unit sphere [S1].
    *   **Background:** Modeled using the same 4D outer volume parameterization and raycasting formulation as NeRF++ [S1].
4.  **Pixel Partitions:** Each training pixel is assigned to the subset of cells whose spatial volume it intersects [S1].

**Algorithm Steps:**

1.  **Spatial Partitioning:**
    Divide the scene volume into $K$ spatial cells $\{V_k\}$ using a geometric clustering algorithm (e.g., Voronoi partitioning) [S1].

2.  **Visibility-Based Data Partitioning:**
    For each training image $i$ with pixel $p$:
    a. Sample points along the camera ray corresponding to pixel $p$.
    b. Identify the set of spatial cells $\mathcal{K}_p = \{k \mid \text{ray}(p) \cap V_k \neq \emptyset\}$ [S1].
    c. Add pixel $p$ to the training set of each submodule $\mathcal{M}_k$ for all $k \in \mathcal{K}_p$ [S1].
    *Note:* This reduces the size of each submodule's trainset significantly (e.g., by 10x) compared to training on all data [S1].

3.  **Foreground/Background Setup:**
    a. Define the foreground volume as an ellipsoid enclosing all camera poses [S1].
    b. Define the background volume as the complement of the foreground [S1].
    c. Use camera altitude measurements to constrain ray sampling bounds, terminating rays near ground level to avoid querying underground regions [S1].

4.  **Parallel Training:**
    Train each submodule $\mathcal{M}_k$ independently and in parallel using its assigned partitioned pixel data [S1]. The loss function is the standard photometric loss between rendered and ground truth colors [S1].

5.  **Rendering (Inference):**
    For a query ray $\mathbf{r}$:
    a. Intersect $\mathbf{r}$ with the partition boundaries (foreground ellipsoid and background sphere).
    b. For each sample point $\mathbf{x}_i$ along $\mathbf{r}$:
        i. Identify the submodule $\mathcal{M}_k$ corresponding to the partition containing $\mathbf{x}_i$.
        ii. Query $\sigma_i = \sigma_k(\mathbf{x}_i, \mathbf{d})$ and $\mathbf{c}_i = \mathbf{c}_k(\mathbf{x}_i, \mathbf{d})$ [S1].
    c. Compute final color $C(\mathbf{r})$ using the standard volumetric rendering integral:
       $$
       C(\mathbf{r}) = \sum_{i} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) \mathbf{c}_i
       $$
       where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1, S3].

6.  **Interactive Rendering Optimization:**
    To ensure near-interactive view generation, exploit temporal coherence by caching occupancy and color values from nearby previous views [S1].

**Parameters:**
*   Number of cells $K$: Not given in the sources.
*   Ellipsoid parameters: Determined by enclosing all camera poses [S1].
*   Altitude threshold: Determined by ground level measurements [S1].

**Edge Cases and Pitfalls:**
*   **Underground Sampling:** Without altitude constraints, rays may query empty underground regions, wasting computation [S1].
*   **Pre-training Infeasibility:** Unlike DeRF or KiloNeRF, MegaNeRF cannot start from a pre-trained large network because training on all available data at this scale is prohibitive [S1].

## Operating characteristics
*   **Training Speed:** Improves training speed by 3x compared to baseline approaches on existing datasets [S1].
*   **Rendering Speed:** Achieves a 40x speedup over conventional NeRF rendering [S1, S3].
*   **Quality:** Remains within 0.8 dB in PSNR quality compared to conventional NeRF [S1, S3]. Improves PSNR by 12% compared to baseline approaches on existing datasets [S1].
*   **Scalability:** Enables data parallelism for large-scale training, scaling to scenes 1000x larger than prior work (150,000 to over 1,300,000 m²) [S1].
*   **Strengths:** Scalable training via data parallelism; interactive rendering via temporal coherence caching; efficient sampling via altitude constraints.
*   **Limitations:** Requires geometric clustering for partitioning; relies on visibility statistics for data assignment.

## Background
MegaNeRF was introduced in 2022 by researchers addressing the challenge of scaling neural radiance fields to massive urban scenes for search-and-rescue applications [S1]. It builds on the visibility statistics analysis of large-scale scenes, noting that only a small fraction of training images are visible from any particular scene point [S1]. It improves upon NeRF++ by using an ellipsoid for the foreground and adding altitude constraints [S1]. It is distinct from DeRF and KiloNeRF in its primary goal of enabling scalable training rather than just fast inference [S1].

## Sources
[S1] Mega-NeRF: Scalable Construction of Large-Scale NeRFs for Virtual Fly- Throughs, https://doi.org/10.1109/cvpr52688.2022.01258, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S3] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
