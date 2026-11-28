# Gradient-Based SE(3) Optimization
Also called: VF-NeRF Optimization
## Overview
Gradient-Based SE(3) Optimization is a method for rigidly registering two radiance fields (Scene A and Scene B) by minimizing the photometric error between them [S1]. It solves the problem of finding the optimal 6-DoF transformation $\hat{T} \in SE(3)$ that aligns a source scene to a target scene [S1]. It differs from single-image alignment methods (like iNeRF) by optimizing over a set of novel views from Scene A, and it differs from unconstrained optimization by using viewshed masks to restrict ray sampling to high-confidence regions, thereby avoiding artifacts from NeRF inaccuracies [S1].
## How it works
### Inputs and Outputs
- **Inputs**:
  - Scene A: A set of novel views $C_A$ and their corresponding viewshed maps (VF masks) [S1].
  - Scene B: A fixed NeRF representation with parameters $\Theta_b$ [S1].
  - Initial transformation estimate $\hat{T}_0 \in SE(3)$ [G].
  - Number of iterations $K$ [G].
  - Batch size $n$ (number of rays per iteration) [S1].
- **Outputs**:
  - The optimized transformation matrix $\hat{T} \in SE(3)$ [S1].

### Data Structures
- **Viewshed Mask**: A binary or continuous mask $M(r)$ associated with each ray $r$ from Scene A, indicating the confidence of the RGB value [S1]. High values indicate reliable geometry/appearance [S1].
- **Ray Set $R_i$**: A set of $n$ rays sampled from the images in $C_A$ at iteration $i$ [S1].
- **Transformation $\hat{T}$**: A $4 \times 4$ homogeneous transformation matrix representing rotation and translation in $SE(3)$ [G].

### Steps
1. **Initialization**: Set $\hat{T} = \hat{T}_0$ [G].
2. **Iteration**: For each iteration $i$ from $1$ to $K$:
   a. **Ray Sampling**: Sample a set of $n$ rays $R_i$ from the images in $C_A$ [S1]. Rays are selected based on high viewshed scores (e.g., thresholding the VF mask) to ensure high confidence in the source RGB values [S1].
   b. **Rendering**: For each ray $r \in R_i$: Render the color $I(r)$ from Scene A (source) [S1]. Apply the current transformation $\hat{T}_{i-1}$ to the ray $r$ to get the transformed ray $r' = \hat{T}_{i-1}(r)$ [S1]. Render the color $I(r')$ from Scene B (target) using the fixed parameters $\Theta_b$ [S1].
   c. **Loss Calculation**: Compute the photometric loss $\mathcal{L}_{photo}$ using the squared difference between the source and transformed target colors [S1]:
      $$
      \mathcal{L}_{photo}(\hat{T}_{i-1}|R_i,\Theta_b) = \frac{1}{n} \sum_{r \in R_i} \left( I(r) - I(\hat{T}_{i-1}(r)) \right)^2
      $$
   d. **Optimization**: Update $\hat{T}$ using Stochastic Gradient Descent (SGD) to minimize $\mathcal{L}_{photo}$ [S1]. The parameters of $\Theta_b$ remain fixed [S1].
3. **Termination**: Return the final $\hat{T}$ that minimizes the photometric loss [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_se3_optimization(scene_a_views, scene_a_masks, scene_b_nerf, T_init, num_iterations, batch_size, lr):
    T = T_init
    for i in range(num_iterations):
        # 1. Sample rays with high viewshed scores
        rays = []
        for view_idx, mask in enumerate(scene_a_masks):
            for ray in scene_a_views[view_idx].rays:
                if mask[ray.pixel] > threshold:
                    rays.append(ray)
        rays = rays[:batch_size]
        
        loss = 0.0
        grad_T = zeros_like(T)
        
        for r in rays:
            # Render source color from Scene A
            color_source = scene_a_views[view_idx].render(r)
            
            # Transform ray to Scene B's coordinate system
            r_transformed = T @ r  # Homogeneous transformation of ray origin/direction
            
            # Render target color from Scene B
            color_target = scene_b_nerf.query(r_transformed)
            
            # Compute squared error
            error = (color_source - color_target) ** 2
            loss += error
            
            # Compute gradients w.r.t T via backpropagation
            grad_T += gradient(error, T)
            
        loss = loss / batch_size
        grad_T = grad_T / batch_size
        
        # 2. Update T using SGD
        T = T - lr * grad_T
        
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
- **Time Complexity**: Linear in the number of iterations and the batch size $n$. Each iteration requires rendering $n$ rays from both Scene A and Scene B [G].
- **Memory Complexity**: Depends on the NeRF representation size and the number of rays sampled [G].
- **Strengths**: Robust to initial pose misalignment due to SGD; avoids NeRF artifacts by using viewshed masks; leverages multiple views for better geometry coverage [S1].
- **Limitations**: Requires fixed NeRF parameters for Scene B; relies on accurate viewshed maps for Scene A [S1].
- **When to Use**: Rigid registration of two NeRFs when a good initial pose is not available and viewshed information is accessible [S1].
- **When Not to Use**: Non-rigid registration; when viewshed maps are unavailable or unreliable [G].

## Background
This method is based on the VF-NeRF paper, which introduces viewshed fields for rigid NeRF registration [S1]. It builds on iNeRF but extends it to multiple images and incorporates viewshed masking [S1]. The method is evaluated on LLFF, object-centric videos, and Objaverse datasets [S1].

## Sources
[S1] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
