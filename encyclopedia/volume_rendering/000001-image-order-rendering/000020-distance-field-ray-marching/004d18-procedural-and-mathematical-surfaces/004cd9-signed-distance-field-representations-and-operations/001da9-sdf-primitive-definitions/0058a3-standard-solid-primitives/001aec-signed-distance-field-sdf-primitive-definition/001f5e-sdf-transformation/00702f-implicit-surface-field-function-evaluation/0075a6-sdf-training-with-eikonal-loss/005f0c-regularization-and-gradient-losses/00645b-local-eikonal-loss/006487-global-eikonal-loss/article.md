# Global Eikonal Loss

Also called: Dense Eikonal Loss, Leik, Leikonal

## Overview
Global Eikonal Loss is a regularization term applied to Signed Distance Field (SDF) representations that enforces the Eikonal equation ($||\nabla f|| = 1$) across a dense grid of samples rather than sparse ray samples [S1, unofficial]. It serves as a dense counterpart to Local Eikonal Loss, ensuring unit gradient magnitude throughout the volume to stabilize SDF training [S1, unofficial]. Unlike ray-based regularizers, it operates on the underlying voxel structure, making its computational cost sensitive to resolution changes [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: SDF network parameters $\theta$, current training iteration $\tau$, and current dense resolution level.
- Output: Scalar loss value $L_{eik}$ added to the total training objective.

**Data Structures**
- **Sample Set $X$**: A set of 3D points consisting of uniformly sampled points within the volume and near-surface points [S2]. The density of $X$ scales with the resolution level [S1, unofficial].

**Parameters**
- Base weight $\lambda_e = 10^{-8}$ [S1, unofficial].
- Decay factor: 0.25 per 2,000 iterations [S1, unofficial].
- Maximum resolution level: 29 [S1, unofficial].
- Maximum iteration count: $\tau = 6,000$ [S1, unofficial].

**Steps**
1. **Check Applicability**: If the current resolution level is greater than 29 or the iteration count $\tau$ exceeds 6,000, return 0 [S1, unofficial].
2. **Sample Points**: Generate the set $X$ of uniformly distributed points and near-surface points [S2].
3. **Evaluate Gradient**: For each point $x \in X$, compute the gradient of the SDF output $f_\theta(x)$ with respect to the input coordinates $x$, denoted $\nabla f_\theta(x)$ [S2].
4. **Compute Eikonal Term**: Calculate the squared deviation from unit magnitude for each point: $(||\nabla f_\theta(x)||_2^2 - 1)^2$ [S2].
5. **Aggregate**: Sum the terms over all $x \in X$ [S2].
6. **Apply Weight**: Multiply the sum by the current weight $\lambda_e(\tau)$ [S1, unofficial].

**Formula**
$$ L_{eik} = \lambda_e(\tau) \sum_{x \in X} \left( ||\nabla f_\theta(x)||_2^2 - 1 \right)^2 $$

Where:
- $X$ is the set of uniformly sampled points together with near-surface points [S2].
- $\nabla f_\theta(x)$ is the gradient of the SDF function at $x$ [S2].
- $\lambda_e(\tau)$ is the time-dependent weight [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def global_eikonal_loss(f_theta, X, tau, current_resolution_level):
    # Step 1: Check applicability
    # Applied only up to resolution level 29, corresponding to tau = 6,000 [S1, unofficial]
    if current_resolution_level > 29 or tau > 6000:
        return 0.0

    # Step 2: Sample points X is assumed to be provided or generated internally
    # X contains uniformly sampled points and near-surface points [S2]

    # Step 3 & 4: Compute Eikonal term for each point
    # Leikonal = sum((||grad||^2 - 1)^2) [S2]
    loss_sum = 0.0
    for x in X:
        # Compute gradient of SDF f_theta at x
        grad = compute_gradient(f_theta, x)
        # Compute squared L2 norm of gradient
        grad_norm_sq = dot(grad, grad)
        # Add Eikonal term
        loss_sum += (grad_norm_sq - 1.0) ** 2

    # Step 5 & 6: Apply weight decay schedule [S1, unofficial]
    # Base weight lambda_e = 10^-8, decays by factor 0.25 every 2,000 iterations
    base_weight = 1e-8
    decay_factor = 0.25
    num_decays = tau // 2000
    current_weight = base_weight * (decay_factor ** num_decays)
    
    return current_weight * loss_sum
```

**Edge Cases and Pitfalls**
- **Resolution Scaling**: The number of samples roughly quadruples whenever the dense resolution increases by one level [S1, unofficial]. This can cause the loss magnitude to spike if not properly weighted [S1, unofficial].
- **Precision**: Computing gradients on dense grids requires efficient automatic differentiation or analytical gradients to avoid excessive memory usage [G].

## Subtopics

### Local Eikonal Loss
Applies Eikonal constraint at voxel centers using analytical gradients of trilinear interpolation [Parent]. Choose when memory is constrained or when sparse sampling is preferred. [Local Eikonal Loss](19082-local-eikonal-loss.md)

### Ray-Eikonal Loss
A ray-based Eikonal regularizer implemented only for the naive baseline SVRaster+SDF, evaluating gradients along ray segments. Choose for ray-marching specific optimizations. [Ray-Eikonal Loss](19083-ray-eikonal-loss.md)

## Operating characteristics
- **Time Complexity**: The computational cost scales with the number of samples in $X$. Since the number of samples roughly quadruples when the dense resolution increases by one level, the cost increases exponentially with resolution [S1, unofficial].
- **Memory Complexity**: Requires storing gradients for all points in $X$ during backpropagation, which scales with the dense grid size [S1, unofficial].
- **Performance**: The loss is annealed to zero in final fine-tuning stages to allow high-fidelity internal losses to define the final geometry [S1, unofficial].
- **Strengths**: Provides dense regularization across the entire volume, stabilizing SDF values [S1, unofficial].
- **Limitations**: High computational cost at high resolutions due to the quadratic increase in samples per resolution level [S1, unofficial].

## Background
Global Eikonal Loss is introduced in the context of neural implicit surface reconstruction, specifically within the MonoSDF framework [S2]. It is designed to regularize SDF values in 3D space by enforcing the Eikonal equation on sampled points [S2]. The method is part of a broader family of Eikonal losses used to improve the geometric accuracy of neural radiance fields and SDFs [S1, unofficial][S2].

## Sources
[S1] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
[S2] MonoSDF: Exploring Monocular Geometric Cues for Neural Implicit Surface Reconstruction, https://doi.org/10.52202/068431-1814, fetched 2026-10-26
