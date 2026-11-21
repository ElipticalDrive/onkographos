# BlitzGS

Also called: BlitzGS: City-Scale Gaussian Splatting at Lightning Speed

## Overview
BlitzGS is a distributed 3D Gaussian Splatting (3DGS) framework designed to accelerate city-scale training by reducing the active Gaussian workload at system, model, and view levels [S1, unofficial]. Unlike spatial block partitioning which suffers from cross-block redundancy and imbalance, BlitzGS shards Gaussians by index parity and couples this with importance-based pruning and distance-based Level-of-Detail (LOD) filtering [S1, unofficial]. It achieves nearly an order-of-magnitude speedup over baselines like Momen.GS and CityGS-X while maintaining high reconstruction quality [S1, unofficial].

## How it works
**Inputs:** A set of calibrated images, an initial set of 3D Gaussians $\mathcal{G}$, and $M$ GPUs.
**Outputs:** Optimized 3D Gaussian parameters (position, scale, rotation, opacity, spherical harmonic coefficients) and a rendered novel view.

**Data Structures:**
*   **Global Gaussian Set $\mathcal{G}$:** Partitioned into shards $\mathcal{G}^{(m)}$ for each GPU $m \in \{0, \dots, M-1\}$.
*   **Sharding Strategy:** Gaussians are assigned to GPU $m$ based on index parity: even indices to one subset of GPUs, odd indices to the other, ensuring balanced distribution without spatial locality constraints [S1, unofficial].
*   **Tile Ownership:** The image plane is divided into tiles. Each tile is assigned to a specific GPU.

**Algorithm Steps:**

1.  **System-Level Sharding and Distribution:**
    *   Partition $\mathcal{G}$ into shards $\mathcal{G}^{(m)}$ such that GPU $m$ stores approximately $|\mathcal{G}|/M$ Gaussians based on index parity [S1, unofficial].
    *   Each GPU $m$ projects its local shard $\mathcal{G}^{(m)}$ to 2D screen space [S1, unofficial].
    *   Perform a single all-to-all communication step to route projected Gaussians to the GPUs responsible for the tiles they cover [S1, unofficial].

2.  **Model-Level Population Control:**
    *   **Importance Scoring:** Compute an importance score for each Gaussian based on its contribution to the loss or gradient magnitude [S1, unofficial].
    *   **Pruning:** Remove Gaussians with low importance scores to reduce redundant primitives [S1, unofficial].
    *   **Spawn Gating:** Before spawning new Gaussians (e.g., via densification), evaluate a lightweight gate. If the candidate is predicted not to survive (low importance), withhold it from spawning [S1, unofficial].
    *   **Gradient Reweighting:** Use the importance signal to reweight density control gradients during optimization [S1, unofficial].

3.  **View-Level Filtering:**
    *   **Distance-Based LOD Gate:** For each camera, drop Gaussians that are too fine (small scale) relative to the camera distance, as they contribute negligible detail [S1, unofficial].
    *   **Importance-Based Mask:** Skip Gaussians with negligible cross-view contribution during rasterization [S1, unofficial].

4.  **Rasterization and Optimization:**
    *   Each GPU composites pixels for its assigned tiles using the received Gaussians [S1, unofficial].
    *   Compute loss and backpropagate gradients.
    *   Update Gaussian parameters using the optimizer state maintained locally on each GPU [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blitzgs_training(images, initial_gaussians, num_gpus, num_iterations):
    # 1. Initialization
    # Shard Gaussians by index parity
    shards = partition_by_parity(initial_gaussians, num_gpus)
    optimizer_states = [create_optimizer_state(shard) for shard in shards]
    
    for iteration in range(num_iterations):
        # 2. System-Level: Projection and Communication
        projected_gaussians = []
        for m in range(num_gpus):
            # Project local shard
            proj = project_2d(shards[m], images[iteration].camera)
            projected_gaussians.append(proj)
        
        # All-to-all exchange: route Gaussians to tile owners
        # Each GPU receives Gaussians covering its assigned tiles
        received_gaussians = all_to_all_exchange(projected_gaussians)
        
        # 3. Model-Level: Population Control
        # Compute importance scores for all active Gaussians
        importance_scores = compute_importance(received_gaussians, optimizer_states)
        
        # Prune low-importance Gaussians
        active_gaussians = prune_by_importance(received_gaussians, importance_scores)
        
        # Spawn gating: evaluate new candidates before adding
        candidates = densification_candidates(active_gaussians)
        surviving_candidates = spawn_gate(candidates, importance_scores)
        active_gaussians = merge(active_gaussians, surviving_candidates)
        
        # 4. View-Level: Filtering
        # Distance-based LOD: drop too-fine Gaussians
        filtered_gaussians = apply_lod_gate(active_gaussians, images[iteration].camera)
        
        # Importance mask: skip negligible contributors
        final_gaussians = apply_importance_mask(filtered_gaussians, importance_scores)
        
        # 5. Rasterization
        # Each GPU renders its tiles
        rendered_images = []
        for m in range(num_gpus):
            # Render tiles owned by GPU m
            tile_images = rasterize_tiles(final_gaussians, images[iteration].camera, m)
            rendered_images.append(tile_images)
        
        # 6. Optimization
        # Compute loss and update
        loss = compute_loss(rendered_images, images[iteration].ground_truth)
        gradients = backward(loss)
        
        # Reweight gradients by importance
        reweighted_gradients = reweight_gradients(gradients, importance_scores)
        
        # Update optimizer states
        for m in range(num_gpus):
            update_optimizer(optimizer_states[m], reweighted_gradients[m])
            shards[m] = optimizer_states[m].get_parameters()
            
    return shards
```

**Parameters:**
*   **Hardware:** 4x NVIDIA A6000 GPUs [S1, unofficial].
*   **Sharding:** Index parity (even/odd) [S1, unofficial].
*   **LOD Threshold:** Not given in the sources.
*   **Importance Threshold:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Imbalanced Shards:** Spatial block partitioning can lead to imbalanced shards due to varying scene density. Index parity sharding avoids this [S1, unofficial].
*   **Cross-Block Visibility:** Spatial partitioning causes visibility waste at block boundaries. Index parity sharding eliminates this need for a final merge step [S1, unofficial].
*   **Precision:** Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Training time is significantly reduced. On the Rubble scene, BlitzGS takes 39 minutes compared to 5 hours 3 minutes for Momen.GS and 4 hours 45 minutes for CityGS-X [S1, unofficial].
*   **Memory Complexity:** By pruning redundant Gaussians and gating spawns, the peak population of Gaussians is limited, reducing memory usage per GPU [S1, unofficial].
*   **Performance:** Achieves scene-average PSNR of 24.66 dB on Residence and 33.79 dB on Rubble [S1, unofficial].
*   **Strengths:** Near order-of-magnitude speedup on city-scale scenes; avoids cross-block redundancy; balanced load distribution.
*   **Limitations:** Requires all-to-all communication; effectiveness depends on the accuracy of the importance scoring and spawn gating [S1, unofficial].
*   **When to use:** Large-scale, city-scale 3DGS training where speed is critical and high-quality reconstruction is required [S1, unofficial].
*   **When not to use:** Small-scale scenes where the overhead of distributed communication and complex filtering may not be justified [S1, unofficial].

## Background
BlitzGS was introduced by Zhongtao Wang, Huishan Au, Yisong Chen, Meng Gai, Yilong Li, Fei Zhu, Mai Su, Haojie Jin, and Guoping Wang in the paper "BlitzGS: City-Scale Gaussian Splatting at Lightning Speed" [S1, unofficial]. The work addresses the high computational cost of city-scale 3DGS training, which involves preprocessing, communication, and rasterization of overly dense primitive sets [S1, unofficial]. It builds on insights from prior work on distributed 3DGS [S1, unofficial].

## Sources
[S1] BlitzGS: City-Scale Gaussian Splatting at Lightning Speed, https://doi.org/10.48550/arXiv.2605.13794, fetched 2026-10-29 (unofficial)
