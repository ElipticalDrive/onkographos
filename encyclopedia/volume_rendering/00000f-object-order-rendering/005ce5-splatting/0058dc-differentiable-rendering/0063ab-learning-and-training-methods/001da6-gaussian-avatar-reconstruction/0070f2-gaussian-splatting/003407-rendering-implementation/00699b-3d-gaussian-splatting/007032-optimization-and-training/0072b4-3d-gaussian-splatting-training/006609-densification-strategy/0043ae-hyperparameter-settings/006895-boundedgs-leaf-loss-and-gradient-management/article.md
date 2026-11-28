# BoundedGS Leaf Loss and Gradient Management

Also called: Adam with Compact Storage

## Overview
BoundedGS Leaf Loss and Gradient Management is a training optimization for 3D Gaussian Splatting that reduces memory overhead during high-resolution rendering by organizing Gaussians into spatial groups (leaves) and using compact gradient storage. It solves the problem of excessive memory consumption caused by maintaining full dense gradients for all Gaussians during the backward pass of large scenes. It differs from sibling methods by explicitly decoupling the gradient accumulation scope to leaf boundaries and using a global ID mapping to feed only necessary gradient rows into the Adam optimizer, while maintaining dense gradients for opacity to support global regularization.

## How it works
**Inputs and Outputs**
- **Inputs**: 3D Gaussian parameters (position, scale, rotation, opacity, spherical harmonic coefficients), camera views, and the current training step.
- **Outputs**: Updated Gaussian parameters via Adam optimizer, densification statistics based on accumulated screen gradients.

**Data Structures**
- **Compact ID Set ($U$)**: A deduplicated union of screened candidate Gaussian IDs across all active groups.
- **ID Map**: An `int32` array of length $N$ (total number of Gaussians) that maps global Gaussian IDs to indices in the compact gradient storage.
- **Gradient Storage**:
    - **Position, SH, Log Scale, Quaternion**: Row-sparse gradients. Only rows corresponding to IDs in $U$ are stored.
    - **Opacity**: Dense gradient storage. All $N$ opacity gradients are maintained because of the global regularization term.
- **Adam State**: Maintained in the standard dense layout (as inherited from ImprovedGS) to ensure compatibility with the optimizer's internal momentum tracking.

**Formulas**
1.  **Leaf Loss ($L_R$)**:
    $$L_R = 0.8 L1_R + 0.2 (1 - SSIM_R)$$
    Where $L1_R$ is the L1 loss and $SSIM_R$ is the Structural Similarity Index Measure loss, computed over the leaf's own image region at its boundary [S1, unofficial].

2.  **View Objective ($L_v$)**:
    $$L_v = \frac{1}{HW} \sum_{R \in R_v} \sum_{P \in P_R} L_R + L_{global}$$
    Where $R_v$ is the set of all leaves for view $v$, $P_R$ are the pixels in leaf $R$, $HW$ is the image height times width, and $L_{global}$ is the global regularization term [S1, unofficial].

3.  **Global Regularization ($L_{global}$)**:
    $$L_{global} = \frac{\beta_t}{|S_t|} \sum_{i \in S_t} z_{it}$$
    Where $S_t$ is the current survivor set of Gaussians, $z_{it}$ is the pre-activation opacity parameter for Gaussian $i$ at step $t$, and $\beta_t$ is the coefficient following GNS's target-budget feedback [S1, unofficial].

**Parameters**
- **Global Regularization Coefficient ($\beta_{15K}$)**: Starts at 0.002 during the 15K–19K pruning window [S1, unofficial].
- **Opacity Learning Rate**: Increases fivefold from 0.025 to 0.125 during the 15K–19K pruning window [S1, unofficial].
- **Adam Optimizer**: $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-15}$ [S1, unofficial].

**Numbered Steps**
1.  **Leaf Loss Computation**: For each leaf $R$ in the current view, compute $L_R$ using L1 and SSIM on the leaf's specific image region [S1, unofficial].
2.  **Global Regularization**: If the current step is within the 15K–19K pruning window, compute $L_{global}$ using the current survivor set $S_t$ and pre-activation opacities $z_{it}$. Add this to the view objective [S1, unofficial].
3.  **Leaf Gradient Accumulation**: For each leaf, compute the image gradient. Explicitly add this contribution to the group’s projected gradients. This is done per leaf [S1, unofficial].
4.  **Gradient Allocation**: Immediately before projection backward, allocate parameter gradients. This step avoids coexistence with the current group’s earlier leaf state [S1, unofficial].
5.  **Projection Backward**: Run projection backward once after all leaves in a group finish. This uses direct gradient accumulation of 3DGS, applies activation derivatives within CUDA, and maps contributions to final candidate gradient rows [S1, unofficial].
6.  **Compact Gradient Mapping**: Feed the compact gradient rows through global IDs to the CUDA Adam optimizer. The ID map links global IDs to compact rows [S1, unofficial].
7.  **Adam Update**: The CUDA Adam updates every Gaussian row. For rows outside the candidates (not in $U$), the current image gradient is zero, but first- and second-order momentum continue to update [S1, unofficial].
8.  **Densification Statistics**: Accumulate screen gradients by global ID across all groups. Use these accumulated gradients to form each Gaussian’s densification statistic [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boundedgs_leaf_loss_and_gradient_management(gaussians, views, step, survivor_set):
    """
    Implements BoundedGS leaf loss and gradient management.
    gaussians: Object containing N Gaussians with attributes:
               pos, sh, log_scale, quat, opacity, adam_m1, adam_m2
    views: List of camera views
    step: Current training step
    survivor_set: Set of Gaussian IDs currently surviving pruning
    """
    # 1. Initialize structures
    # Compact ID set U: deduplicated union of screened candidates across groups
    # ID Map: int32 array of length N, mapping global ID to compact row index
    # Gradient storage: sparse for pos/sh/scale/quat, dense for opacity
    compact_ids = set()
    id_map = {} # global_id -> compact_index
    # ... (Implementation details for building U and id_map based on group screening) ...

    # 2. Process each view
    for view in views:
        view_loss = 0.0
        leaves = get_leaves_for_view(view) # Group Gaussians into spatial leaves
        
        # 2a. Compute Leaf Losses
        for leaf in leaves:
            # Compute L1 and SSIM on leaf's own image region at boundary
            l1_r = compute_l1_loss(leaf.rendered_image, leaf.target_image)
            ssim_r = compute_ssim_loss(leaf.rendered_image, leaf.target_image)
            l_r = 0.8 * l1_r + 0.2 * (1 - ssim_r)
            
            # Add to view objective (area weights restore full-image pixel mean for L1)
            # Note: Summation over pixels P in leaf R is handled internally by rasterization backward
            view_loss += l_r
            
            # 3. Leaf Gradient Accumulation
            # Compute image gradient for this leaf
            leaf_grad = compute_image_gradient(leaf)
            
            # Add contribution to group's projected gradients
            # This is done explicitly per leaf
            add_to_group_projected_gradients(leaf.group_id, leaf_grad)
            
            # Add leaf's Gaussian IDs to compact set
            for gid in leaf.gaussian_ids:
                compact_ids.add(gid)

        # 2b. Global Regularization (if in pruning window)
        if 15000 <= step <= 19000:
            beta_t = 0.002 # Starts at 0.002
            # L_global = (beta_t / |S_t|) * sum(z_it for i in S_t)
            sum_z = sum(gaussians.opacity_pre_activation[i] for i in survivor_set)
            l_global = (beta_t / len(survivor_set)) * sum_z
            view_loss += l_global
            
            # Opacity learning rate adjustment is handled by optimizer config
            # lr_opacity = 0.125 during this window

        # 4. Gradient Allocation
        # Allocate parameter gradients immediately before projection backward
        # to avoid coexistence with current group's earlier leaf state
        allocate_gradients(compact_ids, id_map, gaussians)

        # 5. Projection Backward
        # Run once after all leaves in that group finish
        # Uses direct gradient accumulation of 3DGS
        # Applies activation derivatives within CUDA
        # Maps contributions to final candidate gradient rows
        run_projection_backward(gaussians, id_map, compact_ids)

        # 6. Adam Update
        # Feed compact gradient rows through global IDs to CUDA Adam
        # Adam updates every Gaussian row
        for gid in range(gaussians.N):
            if gid in compact_ids:
                # Get compact gradient row
                compact_idx = id_map[gid]
                grad = get_compact_gradient(compact_idx)
            else:
                # Row outside candidates has zero current image gradient
                grad = 0.0
            
            # Momentum continues to update even if grad is zero
            update_adam_moments(gaussians, gid, grad)
        
        # 7. Densification Statistics
        # View visibility is union of global IDs
        # Screen gradients accumulated by ID across groups
        accumulate_screen_gradients_by_id(gaussians, id_map, compact_ids)
```

**Edge Cases and Pitfalls**
- **Precision**: The use of compact storage requires careful mapping between global IDs and compact rows to avoid misalignment. The `int32` map must be accurate [S1, unofficial].
- **Boundaries**: Leaf loss is computed at the leaf's boundary. Incorrect boundary definition may lead to inaccurate loss gradients [S1, unofficial].
- **Degenerate Input**: If a leaf has no pixels or invalid geometry, the L1/SSIM computation should handle empty regions gracefully [G].
- **Coexistence**: Failing to allocate gradients immediately before projection backward may cause conflicts with earlier leaf states [S1, unofficial].

## Subtopics

### [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md)
Uses conditional logic for optimization steps. Choose when dynamic scene changes require adaptive rendering.

### [Blended Error Densification and Splitting](13892-blended-error-densification-and-splitting.md)
Blends error metrics for densification. Choose when balanced detail and smoothness are needed.

### [Sparse Matching Integration](14995-sparse-matching-integration.md)
Integrates sparse matching for initialization. Choose when sparse data sources are available.

### [Adaptive Gaussian Selector](16104-adaptive-gaussian-selector.md)
Selects Gaussians adaptively. Choose when resource constraints vary per frame.

### [Chunk-scale Training](18156-chunk-scale-training.md)
Trains in chunks. Choose when memory is limited but batch size needs to be large.

### [Edge-guided Splitting Strategy](18680-edge-guided-splitting-strategy.md)
Guides splitting by edges. Choose when sharp features are critical.

### [Standard 3DGS Training Configuration](18796-standard-3dgs-training-configuration.md)
Standard configuration. Choose as a baseline.

## Operating characteristics
- **Time Complexity**: The compact gradient storage reduces the number of gradients processed by Adam, potentially speeding up the update step. However, the overhead of building the compact ID set and mapping may add some CPU time [S1, unofficial].
- **Memory Complexity**: Significantly reduces memory usage for gradient storage by using row-sparse gradients for most parameters. Opacity retains dense gradients, which is a trade-off for global regularization [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales well with high-resolution training due to reduced memory footprint [S1, unofficial].
- **Strengths**: Memory efficiency, compatibility with standard Adam optimizer, support for global regularization.
- **Limitations**: Increased complexity in gradient mapping, potential overhead from ID management.
- **When to use**: High-resolution training, memory-constrained environments.
- **When not to use**: Simple scenes where memory is not a bottleneck, or when global regularization is not desired.

## Background
BoundedGS was introduced in the paper "BoundedGS: Memory-Efficient High-Resolution Training for 3D Gaussian Splatting" (Deng et al., 2025) [S1, unofficial]. It builds on ImprovedGS and GNS (Gaussian Neural Splatting) techniques. The method addresses the memory limitations of standard 3DGS training by organizing Gaussians into leaves and using compact gradient storage.

## Sources
[S1] BoundedGS: Memory-Efficient High-Resolution Training for 3D Gaussian Splatting, https://arxiv.org/abs/2610.34233v1, fetched 2026-10-26 (unofficial)
