# Gradient-Based SE(3) Optimization
Also called: VF-NeRF Optimization
## Overview
Gradient-Based SE(3) Optimization is a method for rigidly registering two radiance fields (Scene A and Scene B) by minimizing the photometric error between them. It solves the problem of finding the optimal 6-DoF transformation $\hat{T} \in SE(3)$ that aligns a source scene to a target scene. It differs from single-image alignment methods (like iNeRF) by optimizing over a set of novel views from Scene A, and it differs from unconstrained optimization by using viewshed masks to restrict ray sampling to high-confidence regions, thereby avoiding artifacts from NeRF inaccuracies [S1].
## How it works
### Inputs and Outputs
- **Inputs**:
  - Scene A: A set of novel views $C_A$ and their corresponding viewshed maps (VF masks).
  - Scene B: A fixed NeRF representation with parameters $\Theta_b$.
  - Initial transformation estimate $\hat{T}_0 \in SE(3)$.
  - Number of iterations $K$.
  - Batch size $n$ (number of rays per iteration).
- **Outputs**:
  - The optimized transformation matrix $\hat{T} \in SE(3)$.

### Data Structures
- **Viewshed Mask**: A binary or continuous mask $M(r)$ associated with each ray $r$ from Scene A, indicating the confidence of the RGB value. High values indicate reliable geometry/appearance.
- **Ray Set $R_i$**: A set of $n$ rays sampled from the images in $C_A$ at iteration $i$.
- **Transformation $\hat{T}$**: A $4 \times 4$ homogeneous transformation matrix representing rotation and translation in $SE(3)$.

### Steps
1. **Initialization**: Set $\hat{T} = \hat{T}_0$.
2. **Iteration**: For each iteration $i$ from $1$ to $K$:
   a. **Ray Sampling**: Sample a set of $n$ rays $R_i$ from the images in $C_A$. Rays are selected based on high viewshed scores (e.g., thresholding the VF mask) to ensure high confidence in the source RGB values [S1].
   b. **Rendering**: For each ray $r \in R_i$:
      - Render the color $I(r)$ from Scene A (source).
      - Apply the current transformation $\hat{T}_{i-1}$ to the ray $r$ to get the transformed ray $r' = \hat{T}_{i-1}(r)$.
      - Render the color $I(r')$ from Scene B (target) using the fixed parameters $\Theta_b$.
   c. **Loss Calculation**: Compute the photometric loss $\mathcal{L}_{photo}$ using the squared difference between the source and transformed target colors [S1]:
      $$
      \mathcal{L}_{photo}(\hat{T}_{i-1}|R_i,\Theta_b) = \frac{1}{n} \sum_{r \in R_i} \left( I(r) - I(\hat{T}_{i-1}(r)) \right)^2
      $$
   d. **Optimization**: Update $\hat{T}$ using Stochastic Gradient Descent (SGD) to minimize $\mathcal{L}_{photo}$. The parameters of $\Theta_b$ remain fixed [S1].
3. **Termination**: Return the final $\hat{T}$ that minimizes the photometric loss.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_se3_optimization(scene_a_views, scene_a_masks, scene_b_nerf, T_init, num_iterations, batch_size, lr):
    """
    Optimizes SE(3) transformation T to align Scene A to Scene B.
    
    Args:
        scene_a_views: List of images from Scene A.
        scene_a_masks: List of viewshed masks corresponding to scene_a_views.
        scene_b_nerf: Fixed NeRF model for Scene B.
        T_init: Initial 4x4 transformation matrix.
        num_iterations: Number of optimization steps.
        batch_size: Number of rays to sample per iteration.
        lr: Learning rate for SGD.
        
    Returns:
        T_opt: Optimized 4x4 transformation matrix.
    """
    T = T_init
    
    for i in range(num_iterations):
        # 1. Sample rays with high viewshed scores
        rays = sample_high_confidence_rays(scene_a_views, scene_a_masks, batch_size)
        
        # 2. Compute photometric loss
        loss = 0.0
        gradients = []
        
        for r in rays:
            # Render source color from Scene A
            color_source = render_ray(r, scene_a_nerf) # Assuming Scene A is also a NeRF or image-based
            
            # Transform ray to Scene B's coordinate system
            r_transformed = apply_transformation(r, T)
            
            # Render target color from Scene B
            color_target = render_ray(r_transformed, scene_b_nerf)
            
            # Compute squared error
            error = (color_source - color_target) ** 2
            loss += error
            
            # Compute gradients w.r.t T (via backprop through render_ray and apply_transformation)
            # Note: In practice, this is done via automatic differentiation
            grad = compute_gradient(error, T)
            gradients.append(grad)
            
        loss = loss / batch_size
        
        # 3. Update T using SGD
        mean_grad = sum(gradients) / len(gradients)
        T = T - lr * mean_grad
        
    return T
```

### Parameters
- **Optimizer**: SGD (Stochastic Gradient Descent) [S1].
- **Learning Rate ($\eta$)**: Not specified in sources [G].
- **Batch Size ($n$)**: Not specified in sources [G].
- **Viewshed Threshold**: Not specified in sources [G].

### Edge Cases and Pitfalls
- **NeRF Artifacts**: Sampling rays in regions with low viewshed scores can lead to artifacts because NeRF may produce unreliable RGB values outside the region of interest. Using thresholded viewshed images mitigates this [S1].
- **Initial Pose**: SGD is used because it is more reliable when the initial transformation $T_0$ is not precise [S1].
- **Multiple Images**: Unlike methods that optimize over a single image, this method samples rays from all images in the set $C_A$, which helps cover scene geometry and improves results [S1].

## Operating characteristics
- **Time Complexity**: Linear in the number of iterations and the batch size $n$. Each iteration requires rendering $n$ rays from both Scene A and Scene B.
- **Memory Complexity**: Depends on the NeRF representation size and the number of rays sampled.
- **Strengths**: Robust to initial pose misalignment due to SGD; avoids NeRF artifacts by using viewshed masks; leverages multiple views for better geometry coverage.
- **Limitations**: Requires fixed NeRF parameters for Scene B; relies on accurate viewshed maps for Scene A.
- **When to Use**: Rigid registration of two NeRFs when a good initial pose is not available and viewshed information is accessible.
- **When Not to Use**: Non-rigid registration; when viewshed maps are unavailable or unreliable.

## Background
This method is based on the VF-NeRF paper, which introduces viewshed fields for rigid NeRF registration. It builds on iNeRF but extends it to multiple images and incorporates viewshed masking [S1]. The method is evaluated on LLFF, object-centric videos, and Objaverse datasets [S1].

## Sources
[S1] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
