# Geometry Consistency Loss

Also called: INS, Self-distilled geometry consistency loss

## Overview
Geometry Consistency Loss is a regularization technique used in differentiable volume rendering, specifically within style transfer frameworks like INS, to enforce view-consistent scene geometry. It addresses the problem of geometric distortions and inconsistencies that arise when applying style transformations to radiance fields by ensuring that the underlying 3D structure remains stable across different viewpoints. Unlike sibling methods such as Curvature Loss or Normal Consistency which focus on local surface properties or alignment with external normals, this method relies on self-distillation to maintain global geometric coherence without requiring explicit depth supervision.

## How it works
The method operates within a representation that is divided into two distinct modules: a style implicit module and a content implicit module, which are fused through an amalgamation module [S1]. The core mechanism involves a self-distillation process where the geometry of the stylized scene is regularized to match the geometry of the original content, ensuring that the style transfer does not alter the underlying 3D structure.

**Inputs and Outputs:**
*   **Inputs:** The radiance field parameters (density $\sigma$, color $\mathbf{c}$), camera poses, and the stylized scene representation.
*   **Outputs:** A scalar loss value $L_{geo}$ used to update the radiance field parameters via gradient descent.

**Data Structures:**
The implementation requires storing the 3D points sampled along rays for both the stylized view ($\hat{p}_t$) and the reference content view ($\hat{p}_{ref}$). These points are typically stored as arrays of coordinates $(x, y, z)$ corresponding to the samples taken during the volume rendering integration [S1].

**Steps:**
1.  **Ray Sampling:** For each pixel in the target stylized view, cast a ray $\mathbf{r}$ through the scene. Sample points along the ray using the standard emission-absorption model defined in [44-radiance-field-optimization.md].
2.  **Point Extraction:** Extract the 3D coordinates of the sampled points $\hat{p}_t^i$ from the stylized radiance field. Simultaneously, extract the corresponding 3D coordinates $\hat{p}_{ref}^i$ from the reference (unstylized) radiance field for the same ray and sample indices [S1].
3.  **Distance Calculation:** Compute the Euclidean distance between corresponding points in the stylized and reference fields.
4.  **Loss Aggregation:** Sum the squared distances over all $N$ correspondences and scale by the hyperparameter $\lambda_{geo}$ [S1].

**Formula:**
The loss is defined as:
$$
L_{geo} = \lambda_{geo} \sum_{i=1}^{N} \|\hat{p}_t^i - \hat{p}_{ref}^i\|_2^2
$$
where:
*   $\hat{p}_t^i$ is the $i$-th 3D point in the stylized scene.
*   $\hat{p}_{ref}^i$ is the corresponding $i$-th 3D point in the reference content scene.
*   $N$ is the total number of correspondences (samples) considered.
*   $\lambda_{geo}$ is a hyperparameter controlling the weight of the geometry loss.

**Parameters:**
*   $\lambda_{geo}$: The weight for the geometry regularization loss. The sources do not specify a default value for the original INS method, but note that in related frequency-guided variants, it is fixed at 0.1 [S2].

**Edge Cases and Pitfalls:**
*   **Misalignment:** If the stylization process causes significant shifts in the perceived depth, the point correspondence $\hat{p}_t^i \leftrightarrow \hat{p}_{ref}^i$ may become invalid if based solely on sample index. The method assumes that the sampling strategy (e.g., uniform or importance sampling) is consistent between the stylized and reference fields [S1].
*   **Memory Constraints:** Storing point coordinates for all rays and samples can be memory-intensive. The original INS paper mentions using deferred back-propagation to optimize with full-resolution images, overcoming GPU memory constraints [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geometry_consistency_loss(stylized_field, reference_field, rays, num_samples):
    """
    Computes the self-distilled geometry consistency loss.
    
    Args:
        stylized_field: Radiance field object for the stylized scene.
        reference_field: Radiance field object for the reference content scene.
        rays: Array of ray origins and directions.
        num_samples: Number of samples per ray.
        
    Returns:
        Scalar loss value.
    """
    # Sample points along rays for the stylized field
    # This returns 3D coordinates for each sample along each ray
    points_stylized = sample_points(stylized_field, rays, num_samples)
    
    # Sample points along rays for the reference field
    # Using the same ray origins and directions ensures correspondence
    points_reference = sample_points(reference_field, rays, num_samples)
    
    # Compute Euclidean distance between corresponding points
    # points_stylized and points_reference have shape (num_rays, num_samples, 3)
    diff = points_stylized - points_reference
    squared_distances = torch.sum(diff ** 2, dim=2) # Sum over x, y, z
    
    # Sum over all samples and rays
    total_loss = torch.sum(squared_distances)
    
    # Scale by hyperparameter lambda_geo
    lambda_geo = 0.1 # Typical value from related variants [S2]
    loss = lambda_geo * total_loss
    
    return loss

def sample_points(field, rays, num_samples):
    """
    Samples 3D points along rays using the field's sampling strategy.
    This is a placeholder for the specific sampling implementation
    (e.g., uniform or importance sampling) used by the radiance field.
    """
    # Implementation depends on the specific radiance field library
    # Returns tensor of shape (num_rays, num_samples, 3)
    pass
```

## Subtopics

### Frequency-Guided Geometry Regularization Loss
A variant of the geometry consistency loss that progressively relaxes geometric constraints over training to mitigate distortions from feature matching errors while leveraging geometric priors in early stages [S2]. It is chosen when sparse matching networks are used for initialization, as it balances the need for initial geometric structure with the flexibility to correct matching errors later [S2]. The loss is defined as:
$$
L_{geo}(\theta, t) = \lambda_{geo} \omega_f(t) \sum_{i=1}^{N} \|\hat{p}_t^i - \hat{p}_{ref}^i\|_2
$$
where $\omega_f(t)$ is a frequency-guided weighting function:
$$
\omega_f(t) = \frac{2 \sum_j M(t,T,L)}{\lambda_f (1 - \sum_j M(0,T,L))}
$$
Here, $M(t,T,L)$ is the frequency mask at iteration $t$, $T$ and $L$ are parameters of the mask, and $\lambda_f$ controls the decay rate [S2]. Parameters: $\lambda_{geo}$ is fixed at 0.1, and $\lambda_f$ is a hyperparameter for decay rate [S2].

## Operating characteristics
*   **Time Complexity:** The loss computation requires sampling points from both the stylized and reference fields, which is $O(N \cdot S)$ where $N$ is the number of rays and $S$ is the number of samples. The distance calculation is linear in the number of samples.
*   **Memory Complexity:** Storing the 3D points for all rays and samples requires $O(N \cdot S \cdot 3)$ memory. This can be significant for high-resolution images and large numbers of samples, necessitating techniques like deferred back-propagation [S1].
*   **Strengths:** It effectively enforces view consistency in stylized scenes without requiring external depth sensors or explicit normal maps. It leverages the self-distillation principle to maintain the original scene's geometry.
*   **Limitations:** It assumes that the reference field's geometry is accurate. If the reference field is poorly reconstructed, the stylized field will inherit these errors. It also requires access to both the stylized and reference fields during training.
*   **When to use:** Use when performing zero-shot 3D style transfer where maintaining the original scene's structure is critical.
*   **When not to use:** Avoid if the reference scene has significant reconstruction artifacts or if computational resources are extremely limited due to memory constraints.

## Background
The Geometry Consistency Loss was introduced in the context of the INS (Implicit Neural Style) framework, which divides the radiance field representation into style and content modules [S1]. It was designed to solve the problem of geometric inconsistency in stylized NeRFs, where style transfer often leads to distorted 3D structures. The method builds on the principles of differentiable volume rendering and self-distillation, where the stylized model is regularized to match the geometry of the original content model [S1]. It is primarily used in 3D style transfer applications [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] SGCNeRF: Few-Shot Neural Rendering via Sparse Geometric Consistency Guidance, https://doi.org/10.1109/tcsvt.2025.3610950, fetched 2026-10-30
