# BlitzGS

Also called: BlitzGS: City-Scale Gaussian Splatting at Lightning Speed
## Overview
BlitzGS is a distributed 3D Gaussian Splatting (3DGS) framework designed to accelerate city-scale reconstruction by reducing the active Gaussian workload at system, model, and view levels [S1, unofficial]. It addresses the high computational cost of training large-scale scenes by avoiding the redundancy and imbalance inherent in spatial block partitioning, achieving nearly an order-of-magnitude faster training times compared to baselines like Momen.GS and CityGS-X while maintaining high reconstruction quality [S1, unofficial].
## How it works
BlitzGS operates on the standard 3DGS primitive definition [21069-rendering-implementation-and-acceleration.md]. It reduces the number of Gaussians processed per iteration through three coupled mechanisms: system-level sharding, model-level population control, and view-level filtering.

### Inputs and Outputs
*   **Inputs:** A set of $N$ 3D Gaussians $\mathcal{G}$, a set of $C$ camera views, and $M$ GPUs.
*   **Outputs:** Rendered images for the camera views and optimized Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients).

### Data Structures
*   **Global Gaussian Set $\mathcal{G}$:** The complete set of Gaussians.
*   **Shard $\mathcal{G}^{(m)}$:** The subset of Gaussians assigned to GPU $m$, where $m \in \{0, \dots, M-1\}$.
*   **Tile Ownership:** The image plane is divided into tiles. Each tile is assigned to a specific GPU.

### Algorithm Steps

1.  **System-Level Sharding (Index Parity):**
    Instead of partitioning Gaussians by spatial location, BlitzGS shards the global set $\mathcal{G}$ across $M$ GPUs based on index parity (even/odd indices) [S1, unofficial].
    *   GPU $m$ stores $\mathcal{G}^{(m)} \approx |\mathcal{G}|/M$ Gaussians and their optimizer states.
    *   This avoids cross-block visibility waste and imbalanced shards associated with spatial partitioning [S1, unofficial].

2.  **Model-Level Population Control:**
    *   **Importance Scoring:** A scoring pass evaluates the importance of each Gaussian. This signal is used to prune redundant Gaussians and reweight density control gradients [S1, unofficial].
    *   **Spawn Gating:** A lightweight gate predicts whether a candidate Gaussian (from densification) will survive. Candidates predicted not to survive are withheld before spawning, limiting peak population [S1, unofficial].

3.  **View-Level Filtering:**
    *   **Distance-based LOD Gate:** Gaussians that are too fine-grained for the current camera distance are dropped [S1, unofficial].
    *   **Importance Mask:** Gaussians with negligible cross-view contribution are skipped, trimming the active set for each camera [S1, unofficial].

4.  **Distributed Rendering Loop:**
    *   **Projection:** Each GPU $m$ projects its local shard $\mathcal{G}^{(m)}$ to screen space.
    *   **Tiling and Routing:** The image is tiled. Each tile is owned by a specific GPU. An all-to-all communication step routes every projected Gaussian to the GPUs owning the tiles it covers [S1, unofficial].
    *   **Rasterization:** Each GPU composites pixels for its owned tiles using the received Gaussians.
    *   **Exchange:** A single cross-GPU exchange ensures each device holds the necessary data for its assigned tiles [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blitzgs_training(gaussians, cameras, num_gpus):
    # 1. System-Level Sharding by Index Parity
    # Split global Gaussian list into M shards based on index parity
    shards = [g[i::num_gpus] for i in range(num_gpus)]
    
    # Assign tiles to GPUs (load balancers ensure balanced assignment)
    tile_owners = assign_tiles_to_gpus(num_gpus)
    
    for iteration in range(max_iterations):
        # 2. Model-Level Population Control
        # Compute importance scores for current Gaussians
        importance_scores = compute_importance(gaussians)
        
        # Prune redundant Gaussians based on importance
        gaussians = prune_gaussians(gaussians, importance_scores)
        
        # Spawn gating: predict survival of new candidates
        # (Implementation detail: [G] specific neural network or heuristic for prediction)
        candidates = densify_gaussians(gaussians)
        surviving_candidates = spawn_gate(candidates, importance_scores)
        gaussians = merge(gaussians, surviving_candidates)
        
        # Reweight density control gradients using importance signal
        # (Implementation detail: [G] gradient modification step)
        
        # 3. View-Level Filtering
        active_gaussians = []
        for cam in cameras:
            # Distance-based LOD gate
            lod_filtered = filter_by_distance(gaussians, cam)
            # Importance-based mask
            view_filtered = filter_by_importance(lod_filtered, cam, importance_scores)
            active_gaussians.append(view_filtered)
            
        # 4. Distributed Rendering
        # Each GPU projects its shard
        projected_shards = [project(g, cam) for g, cam in zip(shards, cameras)]
        
        # All-to-all exchange: route projected Gaussians to tile owners
        # Each GPU sends its projected Gaussians to GPUs owning the tiles they cover
        received_gaussians = all_to_all_exchange(projected_shards, tile_owners)
        
        # Rasterize tiles owned by each GPU
        for gpu_id in range(num_gpus):
            tiles = get_tiles_for_gpu(gpu_id, tile_owners)
            render_tiles(tiles, received_gaussians[gpu_id])
            
        # Backpropagation and optimization step
        compute_loss_and_update(gaussians, shards)
```

### Parameters
*   **Hardware:** 4x NVIDIA A6000 GPUs [S1, unofficial].
*   **Sharding Factor:** $M$ (number of GPUs) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Imbalanced Shards:** Index parity sharding is chosen specifically to avoid the imbalanced shards common in spatial partitioning [S1, unofficial].
*   **Cross-Block Redundancy:** By sharding by index rather than space, cross-block visibility waste is avoided [S1, unofficial].
*   **Merge Step:** Unlike spatial partitioning, BlitzGS does not require a final merge step [S1, unofficial].

## Operating characteristics
*   **Training Time:** BlitzGS trains city-scale scenes in tens of minutes. For the Residence scene, it takes 38 minutes (PSNR 24.66 dB). For the Rubble scene, it takes 39 minutes (PSNR 33.79 dB) [S1, unofficial].
*   **Comparison:** It trains nearly an order of magnitude faster than baselines like Momen.GS (5h 03min, PSNR 31.81) and CityGS-X (4h 45min, PSNR 20.12) [S1, unofficial].
*   **Quality:** Matches recent baselines in rendering quality on large-scale benchmarks (Mill-19, UrbanScene3D, MatrixCity) [S1, unofficial].
*   **Scalability:** Storage and per-step compute scale with the GPU count $M$ [S1, unofficial].
*   **Strengths:** Significant speedup, no merge step, balanced shards.
*   **Limitations:** Requires distributed infrastructure; relies on importance scoring and spawn gating which may have overhead if not lightweight.

## Background
BlitzGS was introduced by Zhongtao Wang, Huishan Au, Yisong Chen, Meng Gai, Yilong Li, Fei Zhu, Mai Su, Haojie Jin, and Guoping Wang in the paper "BlitzGS: City-Scale Gaussian Splatting at Lightning Speed" (arXiv:2605.13794, 2026) [S1, unofficial]. It addresses the challenge of city-scale 3D reconstruction, which requires building scenes quickly, using multiple GPUs efficiently, and rendering high-quality free-viewpoint novel views from thousands of calibrated images [S1, unofficial]. The method builds on insights from prior work on distributed 3DGS [S1, unofficial].

## Sources
[S1] BlitzGS: City-Scale Gaussian Splatting at Lightning Speed, https://doi.org/10.48550/arXiv.2605.13794, fetched 2026-10-29 (unofficial)
