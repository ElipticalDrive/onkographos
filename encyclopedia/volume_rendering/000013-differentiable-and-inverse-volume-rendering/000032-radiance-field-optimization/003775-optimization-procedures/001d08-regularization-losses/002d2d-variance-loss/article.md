# Variance Loss

Also called: Total variance loss, Variance Loss Addition, Variance-based loss, Voxel disparity loss, addVarianceLoss

## Overview
Variance Loss is a regularization technique for differentiable volume rendering that reduces geometric ambiguity in low-texture and few-shot regions by constraining the variance of weight and color distributions along rays [S1, unofficial]. It sets itself apart from standard photometric losses by explicitly penalizing uncertainty in depth estimation and color consistency, thereby encouraging sharper, more compact geometry where gradient signals are weak [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current scene representation (e.g., sparse voxel grid or neural field), camera parameters, and optimization configuration.
*   **Output:** A scalar loss value added to the total optimization objective, which is differentiable with respect to scene parameters (density, color, etc.).

**Data Structures:**
The implementation relies on a configuration structure (`configs`) containing the following fields [S1, unofficial]:
*   `varianceCoefficient` (float): Weight for the variance loss term.
*   `shCoefficient` (float): Weight for the spherical harmonic (SH) loss term.
*   `vairanceType` (int): Switch for loss type. `0` selects Total Variance Loss; any other value selects Voxel Disparity Loss.
*   `neighboorNums` (int): Number of neighbors considered for the variance calculation.
*   `varianceLossInterval` (int): Frequency (in batches) at which the loss is computed.

**Algorithm Steps:**
1.  **Check Interval:** Determine if the current optimization batch should compute the variance loss. This is done by checking if the scene is sparse or if the current batch index modulo the interval equals zero [S1, unofficial]:
    ```python
    should_compute = (not scene.is_sparse) or (batch_index % configs.varianceLossInterval == 0)
    ```
2.  **Select Loss Type:** Based on `configs.vairanceType`, choose between Total Variance Loss and Voxel Disparity Loss [S1, unofficial].
3.  **Compute Loss:** Call the rendering utility function `RenderUtils::addVarianceLoss` passing the scene, optimizer, coefficients, and neighbor count [S1, unofficial].
    *   *Note:* The exact mathematical formulation of the variance calculation (e.g., whether it operates on transmittance weights, density, or color channels) is not explicitly detailed in the source code comments, but it operates on the rendered output or intermediate volume representations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def add_variance_loss(scene, optimizer, configs, batch_index):
    """
    Adds variance-based regularization to the scene optimization.
    Based on implementation in Clear-plenoxels [S1, unofficial].
    """
    # Step 1: Check if variance loss should be computed
    # Condition: Scene is not sparse OR batch index matches interval
    is_sparse = scene.is_sparse
    interval_match = (batch_index % configs.varianceLossInterval) == 0
    
    if is_sparse or interval_match:
        # Step 2: Determine loss type
        # 0 = Total Variance Loss, else = Voxel Disparity Loss
        loss_type = configs.vairanceType
        
        # Step 3: Compute and add loss
        # The actual gradient computation is handled by RenderUtils::addVarianceLoss
        # This function modifies the optimizer's gradient accumulation
        RenderUtils.addVarianceLoss(
            scene=scene,
            optimizer=optimizer,
            variance_coefficient=configs.varianceCoefficient,
            sh_coefficient=configs.shCoefficient,
            loss_type=loss_type,
            neighbor_count=configs.neighboorNums
        )
    else:
        # Skip computation to save resources
        pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** The source code uses `double` precision for camera matrices and intrinsic parameters, suggesting that high precision is required for the underlying geometry, though the variance loss itself may operate on lower precision intermediates [S1, unofficial].
*   **Interval Skipping:** If `varianceLossInterval` is large, the regularization signal is sparse, potentially leading to slower convergence in ambiguous regions [S1, unofficial].
*   **Neighbor Count:** The parameter `neighboorNums` (sic) controls the spatial extent of the variance calculation. An incorrect value may lead to over-smoothing or insufficient regularization [S1, unofficial].

## Subtopics

### [F2-NeRF Training Loss](7645-f2-nerf-training-loss.md)
A composite loss for neural radiance fields with multiresolution hash grids, extending reconstruction loss with disparity regularization and total variation loss for feature consistency.

### [Variance Regularization](9362-variance-regularization.md)
A technique that constrains the variance of weight and color distributions along rays to reduce geometric ambiguity in low-texture regions.

## Operating characteristics
*   **Time Complexity:** Computed only at intervals defined by `varianceLossInterval`. The cost depends on the number of neighbors and the size of the scene representation.
*   **Memory Complexity:** Adds minimal overhead as it operates on existing scene data structures.
*   **Strengths:** Effective at reducing floating artifacts and enforcing spatial smoothness, particularly in unbounded scenes or regions with weak gradient signals [S1, unofficial].
*   **Limitations:** Requires careful tuning of the variance coefficient and interval. May introduce smoothing artifacts if the coefficient is too high.
*   **When to use:** When optimizing radiance fields with sparse data or in unbounded scenes where geometric ambiguity is high.
*   **When not to use:** When computational resources are extremely limited and the interval cannot be reduced, or when the scene is well-constrained with dense multi-view data.

## Background
Variance Loss is implemented in the Clear-plenoxels project, an unofficial extension of Plenoxels [S1, unofficial]. It is designed to improve the stability of depth estimation and the continuity of features in differentiable volume rendering pipelines.

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
