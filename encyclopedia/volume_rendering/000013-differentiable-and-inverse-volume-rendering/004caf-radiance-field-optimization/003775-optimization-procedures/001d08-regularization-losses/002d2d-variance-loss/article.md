# Variance Loss

Also called: Total variance loss, Variance Loss Addition, Variance-based loss, Voxel disparity loss, addVarianceLoss

## Overview
Variance Loss is a regularization technique for differentiable volume rendering that adds a variance-based loss term to scene optimization, selectable as either total variance loss or voxel disparity loss [S1, unofficial]. It builds on [Regularization losses](1413-regularization-losses.md) and sets itself apart from standard photometric losses by explicitly penalizing spatial variance in the rendered output or intermediate volume representations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The scene representation, the scene optimizer, and an optimization configuration structure containing coefficients and parameters [S1, unofficial].
*   **Output:** A scalar loss value added to the total optimization objective, differentiable with respect to scene parameters [G].

**Data Structures:**
The implementation relies on a configuration structure containing the following fields [S1, unofficial]:
*   `varianceCoefficient` (float): Weight for the variance loss term [S1, unofficial].
*   `shCoefficient` (float): Weight for the spherical harmonic (SH) loss term [S1, unofficial].
*   `vairanceType` (int): Switch for loss type where `0` selects Total Variance Loss and any other value selects Voxel Disparity Loss [S1, unofficial].
*   `neighboorNums` (int): Number of neighbors considered for the variance calculation [S1, unofficial].
*   `varianceLossInterval` (int): Frequency (in batches) at which the loss is computed [S1, unofficial].

**Parameters:**
Typical values are not given in the sources [S1, unofficial].

**Algorithm Steps:**
1.  **Check Interval:** Determine if the current optimization batch should compute the variance loss by checking if the scene is not sparse or if the current batch index modulo the interval equals zero [S1, unofficial]:
    ```python
    should_compute = (not scene.is_sparse) or (batch_index % configs.varianceLossInterval == 0)
    ```
2.  **Select Loss Type:** Based on `configs.vairanceType`, choose between Total Variance Loss and Voxel Disparity Loss [S1, unofficial].
3.  **Compute Loss:** Pass the scene, optimizer, coefficients, and neighbor count to the rendering utility function [S1, unofficial].

**Formulas:**
The exact mathematical formulation for the base variance loss is not provided in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The underlying implementation uses `double` precision for camera matrices and intrinsic parameters, suggesting high precision requirements for the base geometry [S1, unofficial].
*   **Interval Skipping:** The loss is only computed when `varianceLossInterval` divides the batch index or the scene is sparse [S1, unofficial].
*   **Neighbor Count:** The parameter `neighboorNums` controls the spatial extent of the variance calculation [S1, unofficial].
*   **Typo:** The source code uses `vairanceType` and `neighboorNums` (sic) [S1, unofficial].
*   Specific effects of coefficient magnitude or interval size on convergence are not detailed in the sources [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def add_variance_loss(scene, optimizer, configs, batch_index):
    """
    Adds variance-based regularization to the scene optimization.
    Based on implementation in Clear-plenoxels [S1, unofficial].
    """
    # Step 1: Check if variance loss should be computed
    is_sparse = scene.is_sparse
    interval_match = (batch_index % configs.varianceLossInterval) == 0
    
    if not is_sparse or interval_match:
        # Step 2: Determine loss type
        # 0 = Total Variance Loss, else = Voxel Disparity Loss
        loss_type = configs.vairanceType
        
        # Step 3: Compute and add loss
        # Note: The exact gradient computation and accumulation logic is not provided in the sources.
        # Standard practice involves computing spatial gradients of the rendered image or volume,
        # calculating variance over neighbor patches, and adding the result to optimizer gradients.
        pass
    # Else: skip computation to save resources
```

## Subtopics

### [F2-NeRF Training Loss](7645-f2-nerf-training-loss.md)
A composite loss for neural radiance fields with multiresolution hash grids, extending reconstruction loss with disparity regularization and total variation loss for feature consistency.

### [Variance Regularization](9362-variance-regularization.md)
A technique that constrains the variance of weight and color distributions along rays to reduce geometric ambiguity in low-texture regions.

### Inter and Intra LoD Variance Measure
A minor variant used in complexity-aware continuous level of detail for Gaussian Splatting [S2, unofficial]. It defines two variance measures for level-of-detail (LoD) transitions: Inter LoD variance $V_{inter}^m = \frac{1}{|P|} \sum_{p \in P} \text{Var}_t [\Delta_t^m(p)]$, which measures whether the same patch degrades smoothly across consecutive LoD transitions, and Intra LoD variance $V_{intra}^m = \frac{1}{T - 1} \sum_{t=1}^{T-1} \text{Var}_{p \in P} [\Delta_t^m(p)]$, which measures whether one LoD transition degrades the image uniformly across patches [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Not given in the sources [S1, unofficial].
*   **Limitations:** Not given in the sources [S1, unofficial].
*   **When to use:** Not given in the sources [S1, unofficial].
*   **When not to use:** Not given in the sources [S1, unofficial].

## Background
Variance Loss is implemented in the Clear-plenoxels project [S1, unofficial].

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S2] KoalaGS: Complexity-Aware Continuous Level of Detail for Gaussian Splatting, https://arxiv.org/abs/2610.28615, fetched 2026-10-26 (unofficial)
