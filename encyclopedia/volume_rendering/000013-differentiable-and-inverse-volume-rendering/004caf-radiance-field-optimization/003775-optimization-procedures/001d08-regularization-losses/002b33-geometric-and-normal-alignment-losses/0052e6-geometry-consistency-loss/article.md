# Geometry Consistency Loss

Also called: INS, Self-distilled geometry consistency loss

## Overview
Geometry Consistency Loss is a regularization technique introduced in the INS framework to enforce view-consistent scene geometry during 3D style transfer [S1]. It divides the radiance field representation into a style implicit module and a content implicit module, which are fused through an amalgamation module [S1]. The method applies a self-distilled loss to regularize the stylized scene's geometry against the original content [S1].

## How it works
**Inputs and Outputs:** The inputs are the stylized and reference radiance field parameters, camera poses, and sampled 3D points; the output is a scalar loss value used for gradient-based optimization [G].

**Data Structures:** The implementation stores 3D points sampled along rays for both the stylized and reference views as coordinate arrays [G].

**Steps:**
1. Sample points along rays for the stylized view.
2. Sample corresponding points along the same rays for the reference content view.
3. Compute the distance between corresponding points.
4. Aggregate the distances and scale by a hyperparameter [G].

**Formula:**
The base formulation computes the distance between corresponding 3D points sampled along rays in the stylized and reference fields [S1]. The explicit mathematical form used in related implementations is:
$$
L_{geo} = \lambda_{geo} \sum_{i=1}^{N} \|\hat{p}_t^i - \hat{p}_{ref}^i\|_2
$$
where:
* $\hat{p}_t^i$ is the $i$-th 3D point in the stylized scene.
* $\hat{p}_{ref}^i$ is the corresponding $i$-th 3D point in the reference content scene.
* $N$ is the total number of correspondences.
* $\lambda_{geo}$ is a hyperparameter controlling the weight of the geometry loss [S2].

**Parameters:**
* $\lambda_{geo}$: Hyperparameter for geometry regularization loss, fixed at 0.1 in experiments [S2].

**Edge Cases and Pitfalls:**
* Misalignment can occur if stylization shifts perceived depth, breaking index-based correspondence between points [G].
* Memory constraints are addressed in the original work by deferring back-propagation to optimize with full-resolution images [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_consistency_loss(points_stylized, points_reference, lambda_geo):
    """
    Computes the self-distilled geometry consistency loss.
    
    Args:
        points_stylized: List of [x, y, z] coordinates for the stylized view.
        points_reference: List of [x, y, z] coordinates for the reference view.
        lambda_geo: Hyperparameter weight for the loss.
        
    Returns:
        Scalar loss value.
    """
    total_distance = 0.0
    for p_t, p_ref in zip(points_stylized, points_reference):
        dx = p_t[0] - p_ref[0]
        dy = p_t[1] - p_ref[1]
        dz = p_t[2] - p_ref[2]
        dist = (dx**2 + dy**2 + dz**2)**0.5
        total_distance += dist
    return lambda_geo * total_distance
```

## Subtopics

### Frequency-Guided Geometry Regularization Loss
A variant that progressively relaxes geometric constraints over training to mitigate distortions from feature matching errors while leveraging geometric priors in early stages [S2]. The loss is defined as:
$$
L_{geo}(\theta, t) = \lambda_{geo} \omega_f(t) \sum_{i=1}^{N} \|\hat{p}_t^i - \hat{p}_{ref}^i\|_2
$$
where $\omega_f(t)$ is a frequency-guided weighting function:
$$
\omega_f(t) = \frac{2 \sum_j M(t,T,L)}{\lambda_f (1 - \sum_j M(0,T,L))}
$$
Here, $M(t,T,L)$ is the frequency mask at iteration $t$, $T$ and $L$ are mask parameters, and $\lambda_f$ controls the decay rate [S2]. Parameters: $\lambda_{geo}$ is fixed at 0.1, and $\lambda_f$ is a hyperparameter for decay rate [S2].

## Operating characteristics
Not given in the sources.

## Background
The Geometry Consistency Loss was introduced in the INS framework, which divides the radiance field representation into style and content modules fused via an amalgamation module [S1]. It was designed to regularize scene geometry and ensure view-consistent stylized scenes [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] SGCNeRF: Few-Shot Neural Rendering via Sparse Geometric Consistency Guidance, https://doi.org/10.1109/tcsvt.2025.3610950, fetched 2026-10-30
