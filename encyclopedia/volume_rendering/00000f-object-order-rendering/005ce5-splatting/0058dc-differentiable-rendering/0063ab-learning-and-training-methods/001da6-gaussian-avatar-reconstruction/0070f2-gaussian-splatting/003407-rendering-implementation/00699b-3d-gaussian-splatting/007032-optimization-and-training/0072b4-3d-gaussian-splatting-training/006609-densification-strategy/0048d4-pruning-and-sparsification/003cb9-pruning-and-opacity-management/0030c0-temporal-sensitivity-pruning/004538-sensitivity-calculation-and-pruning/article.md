# Sensitivity Calculation and Pruning
Also called: Gradient-based Sensitivity Pruning

## Overview
Sensitivity Calculation and Pruning eliminates 3D Gaussians that have no impact on training images by calculating sensitivity via the gradient of total image energy with respect to the parameters [S1]. It differs from [Temporal Sensitivity Pruning](6350-temporal-sensitivity-pruning.md) by using a first-order gradient of total image energy rather than a second-order L2 loss approximation, and from [Gradient-based Pruning](17720-gradient-based-pruning.md) by specifically targeting the elimination of splats with zero sensitivity in color parameters to remove structural redundancy [S1, G].

## How it works
**Inputs:**
*   Set of 3D Gaussians $\mathcal{G}$, where each Gaussian $i$ has parameters including position, covariance, opacity, and spherical harmonic (SH) coefficients for color [G].
*   Training images and their corresponding camera poses [G].

**Outputs:**
*   Pruned set of 3D Gaussians $\mathcal{G}'$ containing only Gaussians with non-zero sensitivity [S1].

**Data Structures:**
*   Gaussian parameters are stored in standard 3DGS format [G].
*   Sensitivity score $S_i$ for each Gaussian $i$, defined as the gradient of total image energy with respect to the parameter [S1].

**Steps:**
1.  **Render Training Images:** Render the current set of Gaussians $\mathcal{G}$ to generate images for all training views [G].
2.  **Calculate Total Image Energy:** Compute the total image energy $E$ across all training views [S1]. The specific formula for energy is not explicitly defined in the sources, but it is derived from the rendered pixel values [S1].
3.  **Compute Sensitivity:** For each Gaussian $i$, calculate its sensitivity $S_i$ using the gradient of the total image energy $E$ with respect to the Gaussian's parameters (specifically the color parameters) [S1].
    *   **Critical Implementation Detail:** When calculating the gradient, **do not clamp** negative direction-dependent colors to zero. The standard 3DGS rendering pipeline clamps negative SH coefficients to zero during forward rendering. However, for sensitivity calculation, these clamped values must be treated as continuous to avoid setting partial derivatives to zero in the backward pass [S1]. If clamping is applied, SH coefficients that contribute to training images will incorrectly yield zero sensitivity [S1].
4.  **Identify Zero-Sensitivity Gaussians:** Identify Gaussians where the sensitivity in the color parameters is exactly zero. The sources observe that up to 15% of Gaussians may exhibit this property [S1].
5.  **Prune:** Eliminate all Gaussians with zero sensitivity from the scene [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_sensitivity_and_prune(gaussians, training_views):
    # Step 1: Render training images without clamping negative direction-dependent colors [S1]
    rendered_images = []
    for view in training_views:
        img = rasterize_gaussians(gaussians, view, clamp_negative_colors=False)
        rendered_images.append(img)
        
    # Step 2: Calculate total image energy E across all training views [S1]
    total_energy = 0.0
    for img in rendered_images:
        for pixel in img:
            for channel in pixel:
                total_energy += channel * channel
                
    # Step 3: Calculate sensitivity Si for each Gaussian i as gradient of E wrt parameters [S1]
    sensitivities = []
    for i, g in enumerate(gaussians):
        grad = compute_gradient_of_energy(total_energy, g, training_views)
        sensitivities.append(grad)
        
    # Step 4: Identify Gaussians with zero sensitivity in color parameters [S1]
    zero_sensitivity_indices = []
    for i, grad in enumerate(sensitivities):
        if is_zero_vector(grad.color_params):
            zero_sensitivity_indices.append(i)
            
    # Step 5: Eliminate zero-sensitivity Gaussians from the scene [S1]
    pruned_gaussians = [g for i, g in enumerate(gaussians) if i not in zero_sensitivity_indices]
    return pruned_gaussians

def rasterize_gaussians(gaussians, view, clamp_negative_colors):
    # Implementation detail [G]: Projects 3D Gaussians to 2D, computes opacity-weighted color accumulation,
    # and applies max(0, c) to color channels only if clamp_negative_colors is True.
    image = empty_image(view.resolution)
    for g in gaussians:
        projected = project_gaussian(g, view.camera)
        color = evaluate_sh(g.sh_coeffs, projected.direction)
        if not clamp_negative_colors:
            color = color # Bypass clamping to preserve gradient flow [S1]
        else:
            color = max(0, color)
        accumulate_to_image(image, projected, color, g.opacity)
    return image

def compute_gradient_of_energy(total_energy, gaussian, views):
    # Implementation detail [G]: Traces the computational graph of the rendering process backwards
    # to compute partial derivatives dE/dparams without zeroing out gradients from clamped values.
    return automatic_differentiation(total_energy, gaussian.params)

def is_zero_vector(vec, threshold=1e-6):
    return sum(x * x for x in vec) < threshold * threshold
```

**Parameters:**
*   Pruning threshold: Not given in the sources. The method primarily targets *zero* sensitivity, but higher thresholds can be used to remove more Gaussians [S1].

**Edge Cases and Pitfalls:**
*   **Clamping Artifacts:** If negative direction-dependent colors are clamped to zero during the backward pass, the partial derivatives become zero, resulting in zero sensitivity for SH coefficients that may still contribute to training images [S1]. This must be avoided.
*   **Loss of Fine Details:** Using higher pruning thresholds (beyond zero) can remove fine details, such as small leaves, from the reconstruction [S1].

## Subtopics

### [Statistical Sensitivity-Driven Online Sparsification](14437-statistical-sensitivity-driven-online-sparsification.md)
Statistical Sensitivity-Driven Online Sparsification is a method for maintaining model compactness in 3D Gaussian Splatting by evaluating the actual contribution of each primitive within the optimization loop. It solves the problem of prohibitive time costs associated with offline pruning by performing sparsification online, ensuring the model remains refined throughout training. It differs from its parent, [Sensitivity Calculation and Pruning](11423-sensitivity-calculation-and-pruning.md), by specifically defining sensitivity as the expected gradient magnitude of the global photometric loss over a fixed accumulation window, rather than a general gradient of total image energy.

### [Gradient-based Pruning](17720-gradient-based-pruning.md)
Gradient-based Pruning is a training-time sparsification technique for 3D Gaussian Splatting that removes Gaussians contributing minimally to the optimization process by evaluating the magnitude of their associated gradients. It distinguishes itself from sibling methods, such as [Statistical Sensitivity-Driven Online Sparsification](14437-statistical-sensitivity-driven-online-sparsification.md), by directly measuring the gradient signal intensity rather than statistical variance or opacity thresholds, allowing for efficient GPU implementation while effectively eliminating visual artifacts like floaters.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The sources note that a notable number of Gaussians (up to 15%) may have no impact on training images and can be safely eliminated [S1].
*   **Strengths:** Removes Gaussians that have no impact on training images, potentially reducing model size without significant loss in PSNR if thresholds are low [S1].
*   **Limitations:** Higher pruning thresholds can lead to the removal of fine details (e.g., small leaves) [S1].
*   **When to use:** When aiming to reduce model size by removing structurally redundant Gaussians that do not contribute to the rendered output [S1].
*   **When not to use:** When preserving fine geometric details is critical, as aggressive pruning may remove them [S1].

## Background
This method is described in "Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis" [S1]. It addresses the issue of redundant Gaussians in 3D Gaussian Splatting by identifying those with zero sensitivity to the total image energy [S1].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
