# Coarse-to-Fine Training Strategy

Also called: Coarse-to-Fine TDBS

## Overview
A training strategy that progressively improves depth precision to circumvent local minima and refine depth geometry [S1]. It sets itself apart by alternating between depth-prior-guided sampling and uniform sampling, whereas its sibling [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md) relies solely on truncated normal distributions throughout training. This alternation accelerates convergence and enhances pose estimation accuracy [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current training epoch $t$, ray $r$, threshold epoch $T_s$.
*   **Output:** A set of sampled points along ray $r$ for volume rendering.

**Data Structures:**
*   $T_s$: Integer threshold epoch.

**Steps:**
1.  Check if the current training epoch $t$ is less than the threshold $T_s$.
2.  If $t < T_s$, execute the **Coarse Step**: Use Truncated Depth-Based Sampling (TDBS) [S1]. This leverages monocular depth priors to converge the density values of the majority of points to optimal values [S1].
3.  If $t \ge T_s$, execute the **Fine Step**: Use Uniform Sampling [S1]. This recomputes erroneous depth values based on the already optimized majority of points, aiming to optimize noisy and erroneous points that might have degraded rendering quality if TDBS were used continuously [S1].

**Formula:**
The sampling strategy $S(r)$ is defined as [S1]:

$$
S(r) = 
\begin{cases} 
S_{TDBS}(r) & \text{if } epoch < T_s \\
S_{uni}(r) & \text{if } epoch \ge T_s 
\end{cases}
$$

Where:
*   $S_{TDBS}(r)$ is the sampling function from [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md).
*   $S_{uni}(r)$ is Uniform Sampling.
*   $T_s$ is the threshold epoch.

**Parameters:**
*   $T_s$: Threshold controlling the switch between sampling methods. Typically set to 1000 epochs, corresponding to the epoch at which the learning rate begins to schedule [S1].

**Edge Cases and Pitfalls:**
*   **Local Minima:** Using full-stage TDBS can cause the model to fall into local minima if the initial depth priors are inaccurate or incorrect [S1]. The coarse-to-fine strategy mitigates this by switching to uniform sampling to correct these errors [S1].
*   **Precision:** The strategy is designed to circumvent local minima and refine depth geometry [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_ray(ray, epoch, threshold_epoch=1000):
    """
    Samples points along a ray using the Coarse-to-Fine strategy.
    
    Args:
        ray: The ray object containing origin and direction.
        epoch: Current training epoch.
        threshold_epoch: The epoch at which to switch from TDBS to Uniform Sampling.
        
    Returns:
        list: A list of sampled 3D points along the ray.
    """
    if epoch < threshold_epoch:
        # Coarse stage: Use Truncated Depth-Based Sampling
        # This relies on the implementation of TDBS to sample based on depth priors
        # See [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md)
        points = sample_tdbS(ray)
    else:
        # Fine stage: Use Uniform Sampling
        # Recompute erroneous depth values based on optimized points
        points = sample_uniform(ray)
        
    return points

def sample_tdbS(ray):
    """
    Placeholder for Truncated Depth-Based Sampling implementation.
    Must be implemented according to [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md).
    """
    # Implementation details are in the parent article
    pass

def sample_uniform(ray):
    """
    Placeholder for Uniform Sampling implementation.
    """
    # Standard uniform sampling along the ray segment
    pass
```

## Subtopics

### Coarse-to-fine Training Strategy for TDBS
A two-stage training strategy that uses TDBS in the coarse stage to converge density values and Uniform Sampling in the fine stage to recompute erroneous depth values and avoid local minima [S1]. It is defined by the formula $S(r) = S_{TDBS}(r)$ if $epoch < T_s$ and $S_{uni}(r)$ if $epoch \ge T_s$ [S1]. The threshold $T_s$ is typically set to 1000 epochs, corresponding to the start of learning rate scheduling [S1].

## Operating characteristics
*   **Convergence:** Accelerates the convergence of pose optimization [S1].
*   **Accuracy:** Enhances the precision of pose estimation [S1].
*   **Robustness:** Avoids degradation of rendering quality caused by inaccurate monocular depth priors by switching to uniform sampling in the fine stage [S1].
*   **Limitations:** Relies on the quality of the initial depth priors for the coarse stage; if priors are severely incorrect, the coarse stage may still introduce noise, though the fine stage mitigates this [S1].

## Background
Proposed in TD-NeRF (2024) as part of a novel truncated depth prior for joint camera pose and neural radiance field optimization [S1]. It addresses the issue where inaccurate monocular depth priors can degrade rendering quality if used throughout the entire training process [S1].

## Sources
[S1] TD-NeRF: Novel Truncated Depth Prior for Joint Camera Pose and Neural Radiance Field Optimization, https://doi.org/10.1109/iros58592.2024.10802634, fetched 2026-10-26
