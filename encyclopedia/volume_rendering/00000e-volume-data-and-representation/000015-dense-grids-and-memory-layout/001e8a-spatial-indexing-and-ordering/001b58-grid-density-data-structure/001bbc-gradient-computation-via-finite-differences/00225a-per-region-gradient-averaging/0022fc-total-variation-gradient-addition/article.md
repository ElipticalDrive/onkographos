# Total Variation Gradient Addition


## Overview
Total Variation Gradient Addition applies a total variation regularization term to the gradient update of specific scalar fields (density, $k_0$, or diffuse) within a grid-based volume representation [S1, unofficial]. It operates as the final optimization step following gradient computation and per-region averaging.

## How it works
**Inputs and Outputs:**
- **Input:** Gradient tensor for a scalar field (density, $k_0$, or diffuse), a scalar weight parameter, the grid's maximum world size, and a boolean `dense_mode` flag.
- **Output:** Modified gradient tensor with the TV regularization term added.

**Data Structures:**
- The field is stored as a dense grid tensor. The `dense_mode` flag controls boundary handling [G].

**Steps:**
1. **Calculate Effective Weight:** Compute a scaled weight $w$ using the input weight and the maximum dimension of the grid.
   $$ w = \text{weight} \times \frac{\max(\text{world\_size})}{128} $$
   The divisor 128 is a fixed scaling constant [S1, unofficial].
2. **Apply TV Gradient:** Add the total variation gradient to the field's gradient tensor using the computed weight and boundary mode [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def total_variation_gradient_add_grad(field_grad, weight, world_size, dense_mode):
    # Step 1: Calculate effective weight
    max_ws = max(world_size) if isinstance(world_size, (list, tuple)) else world_size
    w = weight * (max_ws / 128.0)

    # Step 2: Apply total variation gradient
    # Matches evidence signature: total_variation_add_grad(w, w, w, dense_mode)
    _add_tv_gradient(field_grad, w, w, w, dense_mode)
    return field_grad

def _add_tv_gradient(grad, wx, wy, wz, dense_mode):
    eps = 1e-8
    tv_grad = [0.0] * grad.size
    for axis, w in enumerate([wx, wy, wz]):
        fwd = grad.copy()
        bwd = grad.copy()
        # Compute forward and backward differences along axis
        fwd[axis] = grad[axis + 1:] - grad[:-1]
        bwd[axis] = grad[:-1] - grad[axis + 1:]
        if dense_mode:
            fwd[axis][0] = 0.0
            bwd[axis][-1] = 0.0
        # Subgradient of L1 norm: (fwd + bwd) / (|fwd| + |bwd| + eps)
        mag = fwd[axis] + bwd[axis]
        norm = mag / (abs(fwd[axis]) + abs(bwd[axis]) + eps)
        tv_grad += norm * w
    for i in range(grad.size):
        grad[i] += tv_grad[i]
```

**Parameters:**
- **Weight:** Base regularization strength. Typical values are not specified in the sources.
- **World Size:** Maximum spatial dimension of the grid. Used in weight scaling [S1, unofficial].
- **Dense Mode:** Boolean flag for boundary handling. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Zero Gradient:** If the gradient magnitude is zero, the TV subgradient is undefined. Standard implementations use a small epsilon or default to zero [G].
- **Boundary Conditions:** The `dense_mode` flag controls whether boundaries are clamped to zero or wrapped. Not given in the sources [G].

## Subtopics

### [Total Variation Regularization on Grid](6920-total-variation-regularization-on-grid.md)
Adds a gradient term to the density or embedding grid to encourage piecewise constant solutions, reducing noise and enforcing smoothness. It is applied after gradient computation and per-region averaging, serving as the final step in the optimization update for hash-based or tiled grid encoders.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.

## Background
Total Variation Gradient Addition is part of the optimization pipeline for differentiable volume rendering, specifically in implementations like ref-dvgo [S1, unofficial]. It builds on [Per-Region Gradient Averaging](2715-per-region-gradient-averaging.md) by applying regularization to the averaged gradients.

## Sources
[S1] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
