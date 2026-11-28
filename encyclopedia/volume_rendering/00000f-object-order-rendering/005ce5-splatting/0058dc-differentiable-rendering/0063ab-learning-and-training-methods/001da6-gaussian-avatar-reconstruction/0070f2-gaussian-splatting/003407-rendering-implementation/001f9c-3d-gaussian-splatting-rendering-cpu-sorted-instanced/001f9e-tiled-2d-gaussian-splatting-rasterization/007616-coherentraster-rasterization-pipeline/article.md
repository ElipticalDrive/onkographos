# CoherentRaster Rasterization Pipeline

Also called: CoherentRaster, Efficient 3D Gaussian Splatting for Light Field Displays

## Overview
CoherentRaster is a GPU-parallel rasterization pipeline designed for efficient 3D Gaussian Splatting on light field displays (LFDs), synthesizing interlaced light field images by separating view-dependent attributes from cluster-shared ones [S1]. It distinguishes itself from standard 3D Gaussian Splatting [13469-3d-gaussian-splatting-gpu.md] and tiled variants [2061-tiled-2d-gaussian-splatting-rasterization.md] by employing Cross-view Coherent Attribute Reuse to eliminate redundant projection computations and View-coherent Remapping to enable coalesced memory accesses during alpha blending [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $N$ 3D Gaussians (defined by 3D mean $\mu_i$, covariance $\Sigma_i$, opacity $o_i$, and spherical harmonic coefficients for color); a set of target viewpoints $\mathcal{V} = \{v_j\}_{j=0}^{N-1}$; a tile grid $\mathcal{T}$ (default 32x32 pixels); cluster size $|\mathcal{V}_k|$.
*   **Outputs:** An interlaced light field image $\mathbf{I}_{LF}$ of resolution $W \times H \times 3$ (subpixels).

**Data Structures:**
*   **Viewpoint Clusters:** The viewpoints $\mathcal{V}$ are uniformly partitioned into $K$ disjoint clusters $\{\mathcal{V}_k\}_{k=0}^{K-1}$. Cameras are indexed sequentially along the viewing trajectory and grouped into contiguous clusters. If $N$ is not divisible by cluster size, the final cluster is padded by duplicating the last camera $v_{N-1}$ [S1].
*   **Sorting Key Buffer ($\mathcal{P}$):** Stores tuples of `<key, Gaussian Index i>`.
*   **Sorted List ($\mathcal{L}$):** The result of sorting $\mathcal{P}$, containing Gaussian indices ordered by key.
*   **Attribute Buffers:**
    *   Per-view 2D means: $\mu_{i,j}^{2D}$ for each Gaussian $i$ and view $j$.
    *   Cluster-shared attributes: 2D covariance $\Sigma_{i,k}^{2D}$, depth $d_{i,k}$, and color $\mathbf{c}_{i,k}$ for each Gaussian $i$ and cluster $k$.

**Parameters:**
*   **Bit_K:** The number of bits required to represent the maximum Cluster ID $K$. Not explicitly given in sources, but implied by the key layout formula.
*   **Tile Size:** Default 32x32 pixels, inherited from tiled rasterization [2061-tiled-2d-gaussian-splatting-rasterization.md].

**Numbered Steps:**

1.  **Viewpoint Clustering:** Partition $\mathcal{V}$ into $K$ clusters $\mathcal{V}_k$. Define the cluster center $v'_k$ as the viewpoint at index $\lfloor |\mathcal{V}_k|/2 \rfloor$ within the clustered sequence [S1].

2.  **Stage 1: Projection with Cross-view Coherent Attribute Reuse:**
    *   Initialize buffers for $\mu_{i,j}^{2D}$, $\Sigma_{i,k}^{2D}$, $d_{i,k}$, and $\mathbf{c}_{i,k}$.
    *   **Per-view Means:** Launch $M \times N$ threads (where $M$ is number of Gaussians, $N$ is number of views). Thread $r$ computes $\mu_{i,j}^{2D} = \Pi_{\text{mean}}(v_j; \mu_i)$, projecting the 3D mean to 2D screen space for view $v_j$ [S1].
    *   **Cluster-shared Attributes:** Launch $M \times K$ threads. Thread $r$ computes $\Sigma_{i,k}^{2D}$, $d_{i,k}$, and $\mathbf{c}_{i,k}$ using the cluster center $v'_k$: $\Sigma_{i,k}^{2D}, d_{i,k}, \mathbf{c}_{i,k} = \Pi_{\text{cov,depth,SH}}(v'_k; \Sigma_i, \text{SH}_i)$ [S1]. These attributes are reused for all views $j \in \mathcal{V}_k$.

3.  **Stage 2: Key Generation:**
    *   Initialize empty buffer $\mathcal{P}$.
    *   Launch $M \times K$ threads. Thread $r$ retrieves Gaussian index $i$ and cluster index $k$.
    *   Call `GenerateKeys` (see Subroutine below) to produce keys $K_{i,k}$ and save `<key, i>` to $\mathcal{P}$ [S1].

4.  **Sorting:**
    *   Sort the buffer $\mathcal{P}$ using the generated keys to produce the ordered list $\mathcal{L}$ [S1].

5.  **Stage 4: Alpha Blending with View-coherent Remapping:**
    *   Initialize output image $\mathbf{I}_{LF}$.
    *   Launch $W \times H \times 3$ pixel threads. Thread $r$ obtains remapped coordinate $\hat{x} = \Psi_{\text{remap}}(r)$ [S1].
    *   Identify tile ID $t$ from $\hat{x}$.
    *   Identify per-subpixel viewpoint index $j = V[\hat{x}]$ and cluster ID $k = \text{GetClusterID}(j)$ [S1].
    *   Identify Gaussian list range $[S_{t,k}, E_{t,k}]$ for tile $t$ and cluster $k$ via `IdentifyGaussianList` on $\mathcal{L}$ [S1].
    *   Initialize accumulator $p = 0$.
    *   Loop $e$ from $S_{t,k}$ to $E_{t,k} - 1$:
        *   Retrieve Gaussian index $i = \mathcal{L}[e]$.
        *   Compute $p = \text{BlendColor}(p, \hat{x}, \mu_{i,j}^{2D}, \Sigma_{i,k}^{2D}, \mathbf{c}_{i,k}, o_i)$ [S1].
    *   Set $\mathbf{I}_{LF}[\hat{x}] = p$ [S1].

**Subroutines:**

**GenerateKeys($\mu^{2D}, \Sigma^{2D}, d, \mathcal{T}, r$):**
1.  $i, k \leftarrow \text{GetIndex}(r)$
2.  $\mathcal{T}_{i,k} \leftarrow \emptyset$ // Merge tiles over all views in cluster $\mathcal{V}_k$
3.  For $l$ from 0 to $|\mathcal{V}_k| - 1$:
    a. $j \leftarrow \text{GetViewpointIndex}(k, l)$
    b. $\text{region} \leftarrow \text{ComputeRegionAccuTile}(\mu_{i,j}^{2D}, \Sigma_{i,k}^{2D})$
    c. For each tile $t \in \mathcal{T}$ overlapped by region:
        i. $\mathcal{T}_{i,k} \leftarrow \mathcal{T}_{i,k} \cup \{t\}$
4.  $\mathcal{K}_{i,k} \leftarrow \emptyset$
5.  For each $t \in \mathcal{T}_{i,k}$:
    a. $\text{key} \leftarrow (t \ll (32 + \text{Bit}_K)) \mid (k \ll 32) \mid \overline{d_{i,k}}$
    b. Append `<key, i>` to $\mathcal{K}_{i,k}$
6.  Return $\mathcal{K}_{i,k}$

**Sorting Key Layout:**
The sorting key is constructed as:
$$ \text{key} = (t \ll (32 + \text{Bit}_K)) \mid (k \ll 32) \mid \overline{d_{i,k}} $$
where $t$ is Tile ID, $k$ is Cluster ID, and $\overline{d_{i,k}}$ is the bitwise integer representation of depth $d_{i,k}$ [S1]. This layout sorts Gaussians primarily by spatial location (tile), then by cluster, and finally by depth [S1].

**Edge Cases and Pitfalls:**
*   **Specular Artifacts:** Cross-view Coherent Attribute Reuse can introduce artifacts on highly specular surfaces due to angular deviation within clusters [S1]. This is mitigated in practice by high angular density sampling in commercial Light Field Displays and the natural attenuation of high-frequency specular components by spherical harmonics [S1].
*   **Cluster Padding:** If the number of viewpoints $N$ is not perfectly divisible by the cluster size, the final cluster must be padded by duplicating the last camera $v_{N-1}$ [S1].

## Operating characteristics
*   **Time Complexity:** The method reduces projection, key generation, and sorting times by eliminating redundant computations via attribute reuse. It decreases blending time by enabling coalesced memory accesses via view-coherent remapping [S1].
*   **Memory Complexity:** Significantly reduces VRAM usage compared to baselines without reuse/remapping by storing cluster-shared attributes once per cluster instead of per view [S1].
*   **Measured Performance (RTX 3090, 6MB L2 Cache):**
    *   Synthetic Blender 2K: 36 FPS, 3.01GB VRAM, 15.6M Gaussian-tile pairs, 4.45ms blending time [S1].
    *   Synthetic Blender 4K: 23 FPS [S1].
    *   Mip-NeRF 360 2K: 12 FPS [S1].
    *   Mip-NeRF 360 4K: 6.0 FPS, 5.01GB VRAM, 129.7M Gaussian-tile pairs, 32.41ms blending time [S1].
*   **Comparison:** Outperforms 3DGS (batch=18) which achieves 8.5, 5.4, 3.1, and 1.7 FPS on the respective resolutions [S1]. Compared to 'Ours w/o Reuse w/o Remap' on Synthetic Blender 2K, it reduces VRAM from 4.22GB to 3.01GB and blending time from 15.73ms to 4.45ms [S1]. On Mip-NeRF 360 4K, it reduces VRAM from 18.12GB to 5.01GB and blending time from 67.43ms to 32.41ms [S1].

## Background
CoherentRaster was introduced in the paper "CoherentRaster: Efficient 3D Gaussian Splatting for Light Field Displays" [S1]. It addresses the computational bottleneck of rendering 3D Gaussian Splats for light field displays, which require synthesizing multiple viewpoints per pixel. The method builds on the standard 3D Gaussian Splatting rasterizer [13469-3d-gaussian-splatting-gpu.md] and tiled rasterization techniques [2061-tiled-2d-gaussian-splatting-rasterization.md].

## Sources
[S1] CoherentRaster: Efficient 3D Gaussian Splatting for Light Field Displays, https://doi.org/10.1145/3799902.3811217, fetched 2026-11-08
