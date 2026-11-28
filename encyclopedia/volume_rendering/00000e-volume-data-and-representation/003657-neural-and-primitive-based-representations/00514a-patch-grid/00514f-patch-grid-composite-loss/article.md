# Patch-Grid Composite Loss


## Overview
The Patch-Grid Composite Loss is a composite training objective combining surface, normal, merge, SDF, and Eikonal terms to supervise the neural implicit representation [S1]. It distinguishes itself from standard SDF losses by explicitly including a merge term to constrain the extended zero-level sets of individual patches, ensuring they align correctly at their boundaries for valid CSG operations [S1].

## How it works
The method computes a weighted sum of five distinct loss terms: surface fitting, normal supervision, merge coordination, pseudo-SDF regularization, and Eikonal regularization [S1].

### Inputs and Outputs
*   **Inputs**: Training points sampled from the target surface and its vicinity, ground truth surface positions and normals, neural network parameters, and an octree-based merge grid structure [S1].
*   **Outputs**: A scalar loss value used for gradient-based optimization [S1].

### Data Structures
Standard tensors for points, normals, and network outputs are used [S1]. The merge loss requires an octree-based merge grid to identify leaf nodes where patches overlap or interact [S1].

### Formulas
The total loss is defined as:
$$ L = \lambda_{\text{Surf}} L_{\text{Surf}} + \lambda_{\text{Normal}} L_{\text{Normal}} + \lambda_{\text{Merge}} L_{\text{Merge}} + \lambda_{\text{SDF}} L_{\text{SDF}} + \lambda_{\text{Eikonal}} L_{\text{Eikonal}} $$
Where:
*   $L_{\text{Surf}}$: Surface fitting loss, typically the mean squared error between the predicted SDF value and the ground truth distance (zero for points on the surface) [S1].
*   $L_{\text{Normal}}$: Normal supervision loss, the mean squared error between the normalized gradient of the predicted SDF and the ground truth normal [S1].
*   $L_{\text{Merge}}$: Merge loss, constrains the extended zero-level sets of patches to ensure validity of CSG operations, applied primarily in the leaf nodes of the merge grid [S1].
*   $L_{\text{SDF}}$: Pseudo SDF supervision, applies regularization close to the geometric surface to prevent offsets in individual patches [S1].
*   $L_{\text{Eikonal}}$: Eikonal loss, enforces the unit gradient magnitude property of SDFs: $\|\nabla \phi(p)\|_2 \approx 1$ [S1].

### Parameters
*   $\lambda_{\text{Merge}}$: Default value is 400. Tested range: 100–1200 [S1].
*   $\lambda_{\text{Normal}}$: Default value is 50. Tested range: 12–200 [S1].
*   $\lambda_{\text{Surf}}$, $\lambda_{\text{SDF}}$, $\lambda_{\text{Eikonal}}$: Values not explicitly quantified in the primary source text, though their presence is confirmed by ablation studies [S1].

### Pitfalls and Edge Cases
*   **Low $\lambda_{\text{Merge}}$**: Causes artifacts due to insufficient constraint on coordinating the extraneous zero-level sets, leading to invalid CSG merging [S1].
*   **High $\lambda_{\text{Merge}}$**: Causes holes in the surface due to failure in accurately fitting small, localized patches [S1].
*   **Low $\lambda_{\text{Normal}}$**: Leads to less accurate surface fitting and increased artifacts [S1].
*   **High $\lambda_{\text{Normal}}$**: Overwhelms the merge loss, causing similar artifacts to high $\lambda_{\text{Merge}}$ (failure to handle merging of multiple patches) [S1].
*   **Removing $L_{\text{Surf}}$**: Degrades surface reconstruction quality [S1].
*   **Removing $L_{\text{Normal}}$**: Leads to severe geometric artifacts [S1].
*   **Removing $L_{\text{SDF}}$**: Causes obvious offsets in individual patches and increased Hausdorff distance [S1].
*   **Removing $L_{\text{Eikonal}}$**: Results in a very large Field Error, indicating the learned field does not satisfy the distance field property [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def mean_squared_error(predictions, targets):
    """Computes the mean squared error between two lists of numbers."""
    return sum((p - t) ** 2 for p, t in zip(predictions, targets)) / len(predictions)

def patch_grid_composite_loss(sdf_values, gradients, normals, merge_leaf_mask, sdf_band_mask):
    """
    Computes the composite loss for Patch-Grid training.
    
    Args:
        sdf_values: List of predicted SDF values at query points.
        gradients: List of 3D gradient vectors (dx, dy, dz) at query points.
        normals: List of ground truth surface normal vectors.
        merge_leaf_mask: Boolean list indicating points in leaf nodes of the merge grid.
        sdf_band_mask: Boolean list indicating points in a narrow band around the surface.
        
    Returns:
        total_loss: Scalar loss value.
    """
    # 1. Surface Loss (L_Surf)
    # Supervises SDF values to match ground truth distance (0 for surface points).
    surface_loss = mean_squared_error(sdf_values, [0.0] * len(sdf_values))
    
    # 2. Normal Loss (L_Normal)
    # Supervises gradient direction to match surface normals.
    gradient_magnitudes = [math.sqrt(gx**2 + gy**2 + gz**2) for gx, gy, gz in gradients]
    normalized_gradients = [
        (gx / (mag + 1e-6), gy / (mag + 1e-6), gz / (mag + 1e-6))
        for (gx, gy, gz), mag in zip(gradients, gradient_magnitudes)
    ]
    normal_loss = mean_squared_error(normalized_gradients, normals)
    
    # 3. Merge Loss (L_Merge)
    # Constrains extended zero-level sets for CSG validity in merge grid leaf nodes.
    # [Comment: The exact mathematical formulation and region selection logic are not specified in the sources.]
    merge_values = [v for v, mask in zip(sdf_values, merge_leaf_mask) if mask]
    merge_loss = mean_squared_error(merge_values, [0.0] * len(merge_values)) if merge_values else 0.0
    
    # 4. Pseudo SDF Loss (L_SDF)
    # Applies regularization close to the geometric surface to prevent patch offsets.
    # [Comment: The exact mathematical formulation and band width are not specified in the sources.]
    band_values = [v for v, mask in zip(sdf_values, sdf_band_mask) if mask]
    sdf_loss = mean_squared_error(band_values, [0.0] * len(band_values)) if band_values else 0.0
    
    # 5. Eikonal Loss (L_Eikonal)
    # Enforces unit gradient magnitude property of SDFs.
    eikonal_loss = mean_squared_error(gradient_magnitudes, [1.0] * len(gradient_magnitudes))
    
    # Weights
    lambda_surf = 1.0
    lambda_normal = 50
    lambda_merge = 400
    lambda_sdf = 1.0
    lambda_eikonal = 1.0
    
    total_loss = (lambda_surf * surface_loss + 
                  lambda_normal * normal_loss + 
                  lambda_merge * merge_loss + 
                  lambda_sdf * sdf_loss + 
                  lambda_eikonal * eikonal_loss)
                  
    return total_loss
```

## Subtopics

### Error-driven Surface Fitting for Weight-encoded Neural Implicits
An approach to manage fixed memory cost by scaling network architecture complexity based on input geometry, resulting in smaller architectures for simpler geometries and larger ones for topologically-complex geometries [S2, unofficial]. It performs a simple grid search until reaching a user-desired surface error threshold, and retraining geometries in the tails of the complexity distribution with larger architectures until desired surface fidelity is reached [S2, unofficial]. The base configuration uses 7553 parameters, which lacks representative power for highly complex geometries (similar to a decimated mesh with the same memory footprint), but increasing network capacity to equal the memory impact of the original mesh results in near perfect reconstructions [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of training points. The merge loss is only imposed in the leaf nodes of the merge cells, which account for a small portion of the shape [S1].
*   **Memory Complexity**: Low, as it relies on standard tensors for points, normals, and network outputs, plus the merge grid indices [S1].
*   **Measured Performance**: The complete loss achieves the best accuracy across most metrics (CD, HD, F-score, IoU, NC, SPE, FE) compared to ablated versions [S1].
*   **Strengths**: Effectively coordinates multiple patches for CSG validity; prevents severe artifacts from individual patch failures; ensures smooth, unit-gradient SDF fields [S1].
*   **Limitations**: Sensitive to weight balancing; inappropriate weights for $\lambda_{\text{Merge}}$ or $\lambda_{\text{Normal}}$ lead to visible artifacts (holes or merging failures) [S1].
*   **When to use**: When training Patch-Grid representations that rely on CSG operations to combine local neural SDFs [S1].
*   **When not to use**: For single, global SDF representations where patch merging is not required [S1].

## Background
The Patch-Grid Composite Loss was introduced in the paper "Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation" by Guying Lin et al. [S1]. The method addresses the limitations of uniform patch volumes in fitting thin tubes and complex features by using localized patch-grids. The loss design was validated through ablation studies showing that each term contributes to specific aspects of reconstruction quality, such as surface offset, normal accuracy, and field smoothness [S1].

## Sources
[S1] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
[S2] On the Effectiveness of Weight-Encoded Neural Implicit 3D Shapes, https://doi.org/10.48550/arxiv.2009.09808, fetched 2026-10-27 (unofficial)
