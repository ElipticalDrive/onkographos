# Total Variation Gradient Addition


## Overview
Total Variation Gradient Addition applies a total variation (TV) regularization term to the gradient update of specific scalar fields (such as density, volume density $\sigma$, or diffuse color) within a grid-based volume representation. It serves as the final optimization step for hash-based or tiled grid encoders, following gradient computation and per-region averaging. Unlike standard L2 regularization which penalizes large parameter values, TV regularization penalizes the magnitude of spatial gradients, thereby encouraging piecewise constant solutions that reduce noise while preserving sharp boundaries [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** The current gradient tensor for a specific field (e.g., density, $k_0$, or diffuse) and a scalar weight parameter.
- **Output:** Modified gradient tensor with TV regularization applied.

**Data Structures:**
- The field is stored as a dense grid tensor. The implementation assumes a `dense_mode` flag is available to handle boundary conditions or padding.

**Steps:**
1. **Calculate Effective Weight:** Compute a scaled weight $w$ based on the input weight and the maximum world size of the grid.
   $$ w = \text{weight} \times \frac{\max(\text{world\_size})}{128} $$
   Here, $\max(\text{world\_size})$ is the maximum dimension of the grid in world units or voxel counts, depending on the specific implementation's definition of "world size". The divisor 128 is a fixed normalization constant [S1, unofficial].

2. **Apply TV Gradient:** Add the total variation gradient to the field's gradient tensor. This is typically implemented via a function `total_variation_add_grad(w, w, w, dense_mode)` which computes the divergence of the gradient or the subgradient of the $L_1$ norm of the gradient [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def total_variation_gradient_add_grad(field_grad, weight, world_size, dense_mode):
    """
    Applies total variation regularization to the gradient of a field.
    
    Args:
        field_grad: Tensor of shape (N, ...) representing gradients for the field.
        weight: Scalar weight for the regularization term.
        world_size: Tensor or scalar representing the spatial extent of the grid.
        dense_mode: Boolean flag indicating if dense boundary handling is used.
    
    Returns:
        Modified field_grad tensor.
    """
    # Step 1: Calculate effective weight
    # world_size.max() gets the maximum dimension size
    max_world_size = world_size.max() if hasattr(world_size, 'max') else world_size
    w = weight * (max_world_size / 128.0)
    
    # Step 2: Apply total variation gradient
    # The function total_variation_add_grad is assumed to be defined elsewhere
    # or implemented as a standard TV gradient operator.
    # It modifies field_grad in-place or returns a new tensor.
    # Implementation note: TV gradient usually involves computing differences
    # between neighboring voxels and normalizing by the gradient magnitude.
    field_grad = total_variation_add_grad(w, w, w, dense_mode, field_grad)
    
    return field_grad
```

**Parameters:**
- **Weight:** The base regularization strength. Typical values are not specified in the sources [S1, unofficial].
- **World Size:** The spatial dimensions of the grid. The maximum value is used for normalization [S1, unofficial].
- **Dense Mode:** A boolean flag controlling boundary handling. Not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Gradient:** If the gradient magnitude is zero, the TV subgradient is undefined. Standard implementations use a small epsilon or default to zero [G].
- **Boundary Conditions:** The `dense_mode` flag suggests specific handling for grid boundaries, which may involve padding or clamping [S1, unofficial].

## Subtopics

### [Total Variation Regularization on Grid](6920-total-variation-regularization-on-grid.md)
Adds a gradient term to the density or embedding grid to encourage piecewise constant solutions, reducing noise and enforcing smoothness. It is applied after gradient computation and per-region averaging, serving as the final step in the optimization update for hash-based or tiled grid encoders.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels, as it involves computing differences between neighboring voxels [S1, unofficial].
- **Memory Complexity:** Requires storage for the gradient tensor and potentially temporary buffers for neighbor differences [S1, unofficial].
- **Strengths:** Preserves sharp boundaries while smoothing flat regions, which is beneficial for volume representations with distinct material interfaces [S2, unofficial].
- **Limitations:** May introduce blocky artifacts if the regularization weight is too high [S2, unofficial].

## Background
Total Variation Gradient Addition is part of the optimization pipeline for differentiable volume rendering, specifically in implementations like ref-dvgo. It builds on [Per-Region Gradient Averaging](2715-per-region-gradient-averaging.md) by applying regularization to the averaged gradients. The method is derived from the principles of total variation regularization in image processing and inverse problems [S1, S2, unofficial].

## Sources
[S1] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S2] Total Variation Regularization on Grid, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 [S2, unofficial]
