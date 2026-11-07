# Depth Supervision Loss (Laug)
Also called: Laug
## Overview
Laug is a bidirectional depth consistency loss used to regularize radiance field optimization by enforcing agreement between rendered depth and reliable monocular depth estimates. It distinguishes itself from standard L2 depth losses by incorporating reliability masks and stop-gradient operators to prevent gradient flow into unreliable depth estimates or unstable rendered depths, thereby stabilizing training under sparse input views.
## How it works
**Inputs and Outputs**
- Inputs: Rendered depth map $z_m$, monocular depth estimate $z_a$, reliability masks $m_a$ and $m_m$, and the stop-gradient operator $\hat{\nabla}$.
- Output: Scalar loss value $L_{aug}$.

**Data Structures**
- $z_m$: Rendered depth map, shape $(H, W)$.
- $z_a$: Monocular depth estimate, shape $(H, W)$.
- $m_a$: Mask indicating reliability of monocular depth, binary values $\{0, 1\}$, shape $(H, W)$.
- $m_m$: Mask indicating reliability of rendered depth, binary values $\{0, 1\}$, shape $(H, W)$.

**Steps**
1. Compute the forward consistency term: Calculate the squared L2 distance between the rendered depth $z_m$ and the stop-gradient of the monocular depth $\hat{\nabla}(z_a)$.
2. Compute the backward consistency term: Calculate the squared L2 distance between the stop-gradient of the rendered depth $\hat{\nabla}(z_m)$ and the monocular depth $z_a$.
3. Apply reliability masks: Element-wise multiply the forward term by $m_a$ and the backward term by $m_m$.
4. Sum the terms: Add the masked forward and backward terms to produce $L_{aug}$.

**Formulas**
The loss is defined as [S1]:
$$ L_{aug} = \mathbb{1}_{m_a=1} \odot \| z_m - \hat{\nabla}(z_a) \|^2 + \mathbb{1}_{m_m=1} \odot \| \hat{\nabla}(z_m) - z_a \|^2 $$
Where:
- $\odot$ denotes element-wise multiplication.
- $\mathbb{1}_{condition}$ is the indicator function, returning 1 if the condition is true, 0 otherwise.
- $\hat{\nabla}(\cdot)$ is the stop-gradient operator, which blocks gradient propagation through its argument during backpropagation.
- $\| \cdot \|^2$ is the squared L2 norm (element-wise squared difference).

**Parameters**
- Typical values: Masks $m_a, m_m$ are binary. No other hyperparameters are specified in the source.

**Edge Cases and Pitfalls**
- Precision: The stop-gradient operator is critical. Without it, gradients from the loss may flow into the monocular depth estimator or the depth renderer, potentially destabilizing the optimization [S1].
- Boundaries: The reliability masks must be correctly computed to exclude invalid depth regions (e.g., occlusions or low-confidence areas) to avoid introducing noise into the radiance field [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_laug_loss(z_m, z_a, m_a, m_m):
    """
    Compute the bidirectional depth supervision loss.
    
    Args:
        z_m: Rendered depth map (H, W)
        z_a: Monocular depth estimate (H, W)
        m_a: Reliability mask for monocular depth (H, W), binary
        m_m: Reliability mask for rendered depth (H, W), binary
        
    Returns:
        Scalar loss value
    """
    # Stop-gradient operator blocks gradients
    # Note: In practice, this is implemented using framework-specific 
    # stop_gradient or detach() functions.
    z_a_stopped = stop_gradient(z_a)
    z_m_stopped = stop_gradient(z_m)
    
    # Forward term: Rendered vs Monocular (monocular stopped)
    forward_diff = z_m - z_a_stopped
    forward_loss = forward_diff ** 2
    
    # Backward term: Monocular vs Rendered (rendered stopped)
    backward_diff = z_m_stopped - z_a
    backward_loss = backward_diff ** 2
    
    # Apply reliability masks
    # m_a and m_m are binary masks (0 or 1)
    masked_forward = forward_loss * m_a
    masked_backward = backward_loss * m_m
    
    # Sum and average over pixels (optional, depending on implementation)
    # The source formula implies element-wise sum, but typically reduced to scalar
    total_loss = masked_forward + masked_backward
    
    # Return mean loss over valid pixels or total sum
    # Source does not specify reduction, assuming sum over all elements for L2 norm definition
    return total_loss.sum()
```

## Subtopics

### Reliability Masking
Determines which depth pixels are trustworthy. It sets apart Laug from naive depth losses by ignoring unreliable estimates. Choose when input views are sparse and monocular depth is noisy. [Reliable Depth Estimation via Reprojection](7293-reliable-depth-estimation-via-reprojection.md)

## Operating characteristics
- Time and memory complexity: Linear in the number of pixels $H \times W$. The stop-gradient operation adds negligible overhead.
- Measured performance: Effective in regularizing NeRF, TensoRF, and ZipNeRF under sparse input conditions [S1].
- Scaling: Scales linearly with resolution.
- Strengths: Stabilizes training by preventing gradient flow into unreliable depth estimates.
- Limitations: Depends entirely on the quality of the reliability masks.
- When to use: Sparse input view reconstruction where monocular depth is available but noisy.
- When not: Dense input views where depth supervision is less critical.

## Background
Originated in "Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions" by [S1]. It is part of a family of augmentations for simple radiance fields to handle sparse inputs.

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
