# Macro-tile Rasterization

Also called: Inline visibility macro-tile rasterization

## Overview
Macro-tile Rasterization is a GPU-accelerated technique for 3D Gaussian Splatting that replaces the standard four-stage per-tile gaussian list materialization (bitmask generation, counting, prefix sum, fill) with inline visibility determination during the gaussian load phase [S1, unofficial]. It decouples work distribution from screen area by grouping render tiles into macro-tiles (8x4 or 8x8), bounding per-unit work to achieve resolution-stable frame times [S1, unofficial]. Unlike baseline rasterizers where partitioning and sorting dominate frame time at high resolutions, HiGS reduces the partitioning bottleneck by ensuring the macro-tile pair count grows significantly slower than screen resolution [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of $B_r$ Gaussians, a set of $M$ render tiles, a tile work queue $Q$, and a processing unit configuration of $W$ worker groups with $P$ threads each.
*   **Output:** Final pixel colors for all render tiles, computed via front-to-back alpha compositing.

**Data Structures:**
*   **Local Storage ($\mathcal{S}$):** Holds gaussian data for the current batch.
    *   $\mathcal{S}.\text{pos}[j]$: Position of gaussian $j$.
    *   $\mathcal{S}.\text{conic}[j]$: Conic matrix of gaussian $j$.
    *   $\mathcal{S}.\text{color}[j]$: Color of gaussian $j$.
    *   $\mathcal{S}.\text{masks}[k \cdot M + \text{thread\_id}]$: Stores transposed masks. Index $k$ is the mini-batch index, $\text{thread\_id}$ is the thread within the worker group.
*   **Scheduling Buffer ($\mathcal{R}$):** Capacity $2B_d$. Decouples variable-rate mask compaction from fixed-rate blending.
*   **Tile Work Queue ($Q$):** Contains active render tiles assigned to worker groups.

**Parameters:**
*   $B_r$: Total number of Gaussians in the batch.
*   $M$: Number of render tiles.
*   $P$: Number of threads per worker group.
*   $W$: Number of worker groups per processing unit.
*   $K = \lceil B_r / P \rceil$: Number of mini-batches.
*   $B_d$: Fixed-size drain batch size for blending.
*   Precision: Half-precision (fp16) is used for memory efficiency on commodity GPUs, as packed half-precision and single-precision share resources [S1, unofficial].

**Steps:**

1.  **Phase 1: Load and Inline Visibility**
    The processing unit cooperatively loads its batch of $B_r$ Gaussians into local storage in $K$ mini-batches of $P$ Gaussians, one per thread of a worker group. During the load, each Gaussian is tested against all $M$ render tiles using a conservative overlap test on its opacity-threshold ellipse, producing a per-gaussian tile-visibility mask. These masks are transposed from per-gaussian to per-tile format.

    *   Compute $K \leftarrow \lceil B_r / P \rceil$.
    *   For each mini-batch $k$ from $\text{group\_id}$ to $K-1$ with stride $W$:
        *   Compute global gaussian index $j \leftarrow k \cdot P + \text{thread\_id}$.
        *   Load Gaussian data: $\mathcal{S}.\text{pos}[j], \mathcal{S}.\text{conic}[j], \mathcal{S}.\text{color}[j] \leftarrow \text{LoadGaussian}(mt, j)$.
        *   Test overlap: $\text{tile\_mask} \leftarrow \text{OverlapTest}(\mathcal{S}.\text{pos}[j], \mathcal{S}.\text{conic}[j])$. This produces an $M$-bit mask where bit $t$ is set if tile $t$ overlaps the Gaussian.
        *   Transpose mask: $\mathcal{S}.\text{masks}[k \cdot M + \text{thread\_id}] \leftarrow \text{TransposeMasks}(\text{tile\_mask})$. This converts the per-gaussian mask into a per-tile visibility entry for this mini-batch.
    *   Synchronize all worker groups.
    *   If $\text{group\_id} = 0$:
        *   Compute valid tile mask: $\text{hit} \leftarrow \bigvee_{k=0}^{K-1} \mathcal{S}.\text{masks}[k \cdot M + \text{thread\_id}]$. This OR-reduce determines which tiles have at least one visible Gaussian in the batch.

2.  **Phase 2: Per-tile Rasterization**
    Active render tiles are assigned to worker groups via the tile work queue $Q$. For each claimed tile, the worker group iterates over the precomputed mask, extracting only the visible Gaussians from local storage. A scheduling buffer $\mathcal{R}$ accumulates Gaussian indices from the mask, then drains them in fixed-size drain batches of $B_d$ Gaussians through an unrolled blending core. This decouples variable-rate mask iteration from fixed-rate blending arithmetic. Blending uses front-to-back alpha compositing with half-precision accumulators tracking per-pixel RGBT. After each drain batch, a unanimous vote tests whether all pixels in the tile have reached transmittance saturation; if so, the worker group immediately claims the next tile from the queue.

    *   Claim a tile from $Q$.
    *   Iterate over the precomputed mask for the claimed tile to extract visible Gaussian indices from $\mathcal{S}$.
    *   Accumulate indices in scheduling buffer $\mathcal{R}$.
    *   While $\mathcal{R}$ is not empty:
        *   Drain $B_d$ Gaussians from $\mathcal{R}$.
        *   Perform $\text{FrontToBackBlend}$ on these $B_d$ Gaussians using half-precision accumulators.
        *   Test for transmittance saturation: If all pixels in the tile have $\prod T_i < \epsilon$, break and claim next tile.

3.  **Post-rasterization Compositing**
    Because each macro-tile's depth-sorted batches are processed sequentially, the final color is accumulated across batches.

    *   Walk each render tile's batch chain in depth order.
    *   Accumulate color using front-to-back alpha compositing with transmittance early-out:
        $$C = C_0 + T_0 \cdot C_1 + T_0 T_1 \cdot C_2 + \cdots$$
        with early termination when $\prod_i T_i < \epsilon$.
    *   Each processing unit writes partial RGBT only for tiles with visible Gaussians using a sparse buffer indexed by a per-batch active-tile bitmask.
    *   Worker group reads compact stream of partial RGBT records.
    *   Tiles that saturate early skip remaining batches.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Algorithm 1: Phase 1 Loop
def phase_1_load_and_inline_visibility(B_r, M, W, P, group_id, thread_id):
    K = ceil(B_r / P)
    # Local storage for this processing unit
    S_pos = [None] * B_r
    S_conic = [None] * B_r
    S_color = [None] * B_r
    S_masks = [0] * (K * M)  # Transposed masks

    # Stride over mini-batches
    for k in range(group_id, K, W):
        j = k * P + thread_id
        if j < B_r:
            # Load Gaussian data
            S_pos[j], S_conic[j], S_color[j] = LoadGaussian(mt, j)
            
            # Test gaussian against all M render tiles
            tile_mask = OverlapTest(S_pos[j], S_conic[j])
            
            # Transpose masks from per-gaussian to per-tile
            # tile_mask is an M-bit integer; TransposeMasks returns the transposed representation
            S_masks[k * M + thread_id] = TransposeMasks(tile_mask)

    # Barrier synchronization
    synchronize()

    # Compute valid tile mask if this is the first group
    if group_id == 0:
        hit = 0
        for k in range(K):
            hit |= S_masks[k * M + thread_id]
        return hit, S_pos, S_conic, S_color, S_masks
    return None, S_pos, S_conic, S_color, S_masks

# Algorithm 2: Core Loop
def phase_2_per_tile_rasterization(tile, S_pos, S_conic, S_color, S_masks, B_d):
    # Extract visible gaussians for the claimed tile from precomputed mask
    # The mask for tile t is stored in S_masks, but needs to be reconstructed
    # from the per-mini-batch transposed masks.
    # For simplicity, assume we have a function to get visible indices for a tile
    visible_indices = GetVisibleIndices(tile, S_masks)
    
    # Scheduling buffer R
    R = []
    
    # Accumulate indices
    for idx in visible_indices:
        R.append(idx)
        
        # Drain in fixed-size batches
        if len(R) >= B_d:
            batch = R[:B_d]
            R = R[B_d:]
            
            # Unrolled blending core
            FrontToBackBlend(batch, S_pos, S_conic, S_color)
            
            # Test for transmittance saturation
            if IsTileSaturated(tile):
                return # Skip remaining batches for this tile

    # Drain remaining
    if R:
        FrontToBackBlend(R, S_pos, S_conic, S_color)

def FrontToBackBlend(batch, S_pos, S_conic, S_color):
    # Accumulate Eq. (1) front-to-back with transmittance early-out
    # Each processing unit writes partial RGBT only for tiles with visible gaussians
    # using a sparse buffer indexed by a per-batch active-tile bitmask.
    for idx in batch:
        # Perform half-precision front-to-back alpha compositing
        # C = C_0 + T_0 * C_1 + ...
        CompositePixel(S_pos[idx], S_conic[idx], S_color[idx])

def IsTileSaturated(tile):
    # Check if all pixels in the tile have reached transmittance saturation
    # i.e., product of transmittances < epsilon
    return tile.transmittance_product < epsilon
```

## Subtopics

### [HiGS Macro-tile Rasterization](23058-higs-macro-tile-rasterization.md)
HiGS Macro-tile Rasterization is a GPU-accelerated rasterization technique for 3D Gaussian Splatting that replaces standard per-tile bitmask generation and prefix-sum dispatch with inline visibility determination during gaussian loading. It decouples work distribution from screen area by grouping render tiles into macro-tiles (8x4 or 8x8), bounding per-unit work to achieve resolution-stable frame times. Unlike baseline rasterizers where partitioning and sorting dominate frame time at high resolutions, HiGS reduces the partitioning bottleneck by ensuring the macro-tile pair count grows significantly slower than screen resolution.

## Operating characteristics
*   **Time Complexity:** The method replaces the $O(N \log N)$ or $O(N)$ sorting and prefix-sum stages of standard rasterizers with an $O(B_r \cdot M)$ overlap test phase, where $B_r$ is the batch size and $M$ is the number of tiles. However, the overlap test is done inline and in parallel, and the subsequent blending is $O(N)$ where $N$ is the number of visible Gaussians. The key benefit is that the partitioning bottleneck is reduced because the macro-tile pair count grows significantly slower than screen resolution [S1, unofficial].
*   **Memory Complexity:** Local storage $\mathcal{S}$ holds $B_r$ Gaussians and $K \cdot M$ mask entries. The scheduling buffer $\mathcal{R}$ has capacity $2B_d$. Half-precision is used for memory efficiency [S1, unofficial].
*   **Measured Performance:** HiGS achieves 2.5–4.2 ms for a global radix sort over the same content, indicating significant savings compared to traditional sorting-based approaches [S1, unofficial].
*   **Strengths:** Decouples variable-rate mask iteration from fixed-rate blending, keeping utilization high regardless of per-tile density. Reduces partitioning bottleneck at high resolutions.
*   **Limitations:** Requires conservative overlap test on opacity-threshold ellipse, which may include some false positives. The inline visibility determination requires careful synchronization and mask transposition.
*   **When to use:** When rendering 3D Gaussian Splatting at high resolutions where traditional sorting and prefix-sum stages become a bottleneck.
*   **When not to use:** When the number of Gaussians is very small, or when the overhead of inline visibility determination outweighs the benefits of avoiding sorting.

## Background
Macro-tile Rasterization was introduced in the HiGS paper as a novel approach to 3D Gaussian Splatting rasterization [S1, unofficial]. It addresses the limitations of standard rasterizers that rely on explicit per-tile gaussian lists, which require costly bitmask generation, counting, prefix sum, and fill stages. By moving visibility determination to the load phase and using macro-tiles to group work, HiGS achieves better scalability and performance on commodity GPUs.

## Sources
[S1] HiGS: A Hierarchical Rendering Architecture for Real-Time 3D Gaussian Splatting, https://doi.org/10.48550/arXiv.2606.00352, fetched 2026-10-29 (unofficial)
