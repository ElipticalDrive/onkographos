# Gaussian Primitive Training Schedule

Also called: Reduced Iteration Schedule, Two-Phase Training Schedule

## Overview
This method reduces the training iteration count of 3D Gaussian Splatting from the conventional ~300,000 steps to approximately 6,000 steps by leveraging dense initial point cloud initialization [S1]. It achieves satisfactory reconstruction quality significantly faster than standard baselines by splitting the optimization into two short phases separated by pseudo-view depth refinement [S1].

## How it works
**Inputs:**
- A dense initial point cloud (e.g., from DUSt3R or similar dense reconstruction) [S1].
- A set of input training views with known camera poses [S1].
- Hyperparameters: `total_iterations = 6000`, `phase1_iterations = 3000`, `phase2_iterations = 3000`, `densify_interval = 500`, `densify_skip = 1500` [S1].

**Outputs:**
- Optimized 3D Gaussian primitives (position, scale, rotation, opacity, spherical harmonics coefficients) [S1].

**Data Structures:**
- `Gaussians`: List of primitive objects. Each primitive stores:
  - `position`: 3D vector $(x, y, z)$
  - `scale`: 3D vector $(s_x, s_y, s_z)$
  - `rotation`: Quaternion $(q_w, q_x, q_y, q_z)$
  - `opacity`: Scalar $\alpha \in [0, 1]$
  - `sh_coeffs`: Spherical Harmonics coefficients for color [S1].

**Algorithm Steps:**
1. **Initialization:** Initialize the Gaussian primitives using the dense point cloud. Set the current iteration counter $t = 0$ [S1].
2. **Phase 1 Training:**
   - For $t$ from 0 to 2999:
     - Sample a random training image and camera pose [S1].
     - Render the image $\hat{I}$ using the differentiable rasterizer [S1].
     - Compute loss $\mathcal{L} = (1 - \lambda) \mathcal{L}_{L1} + \lambda \mathcal{L}_{D-SSIM}$ with $\lambda = 0.2$ [S1].
     - Backpropagate gradients and update parameters using Adam ($\beta_1=0.9, \beta_2=0.99$) [S1].
     - If $t > 1500$ and $t \pmod{500} == 0$:
       - Apply densification strategy (splitting high-gradient Gaussians, pruning low-opacity ones) as defined in AbsGS [S1].
3. **Pseudo-View Depth Optimization:**
   - Render RGB images for "pseudo views" (synthetic views generated from the current Gaussian representation) [S1].
   - Perform depth optimization specifically for these pseudo views to refine geometric consistency [S1].
4. **Phase 2 Training:**
   - For $t$ from 3000 to 5999:
     - Sample a random training image and camera pose [S1].
     - Render the image $\hat{I}$ [S1].
     - Compute loss $\mathcal{L}$ [S1].
     - Backpropagate and update parameters [S1].
     - If $t \pmod{500} == 0$:
       - Apply densification strategy [S1].
5. **Completion:** Return the optimized Gaussians [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_gaussian_schedule(gaussians, training_views, dense_init_points):
    # Initialize Gaussians from dense point cloud [S1]
    gaussians = initialize_from_points(dense_init_points)
    
    total_iterations = 6000
    phase1_end = 3000
    densify_skip = 1500
    densify_interval = 500
    
    for t in range(total_iterations):
        # 1. Sampling [S1]
        view_idx = random_choice(training_views)
        camera_pose = training_views[view_idx].pose
        gt_image = training_views[view_idx].image
        
        # 2. Rendering [S1]
        rendered_image = render(gaussians, camera_pose)
        
        # 3. Loss Computation [S1]
        # Standard 3DGS loss: L_img = (1 - lambda) * L1 + lambda * D-SSIM
        lambda_weight = 0.2
        loss = compute_loss(rendered_image, gt_image, lambda_weight)
        
        # 4. Backpropagation & Update [S1]
        gradients = compute_gradients(loss, gaussians)
        update_parameters(gaussians, gradients, optimizer="Adam", beta1=0.9, beta2=0.99)
        
        # 5. Densification/Pruning [S1]
        # Skip first 1500 iterations, then every 500 iterations
        if t > densify_skip and (t - densify_skip) % densify_interval == 0:
            apply_densification(gaussians, gradients)
            
        # Special Step: Pseudo-View Depth Optimization after Phase 1 [S1]
        if t == phase1_end:
            pseudo_views = generate_pseudo_views(gaussians)
            for p_view in pseudo_views:
                p_gt_image = render(p_view.camera_pose, gaussians)
                # Depth optimization loop for pseudo views
                optimize_depth_for_pseudo_view(gaussians, p_view)
                
    return gaussians
```

**Edge Cases and Pitfalls:**
- **Initialization Density:** The method relies on "sufficiently dense point clouds" for initialization [S1]. Sparse initialization may not converge within 6,000 iterations.
- **Densification Timing:** Applying densification too early (before iteration 1500) is explicitly avoided to prevent instability during the initial convergence phase [S1].
- **Pseudo-View Generation:** The quality of the "pseudo views" and the subsequent depth optimization step is critical for the second phase's success [S1].

## Subtopics

## Operating characteristics
- **Time Complexity:** Significantly reduced compared to standard 3DGS. Requires ~6,000 iterations versus ~300,000 for conventional methods [S1]. This represents a ~50x reduction in training time.
- **Memory Complexity:** Similar to standard 3DGS, as the number of Gaussians is determined by the initialization density and densification strategy, not the iteration count [S1].
- **Performance:** Achieves "satisfactory results" in the first 3,000 iterations [S1].
- **Hardware:** Experiments conducted on NVIDIA RTX 4090 GPU [S1].
- **Strengths:** Drastic reduction in training time; suitable for applications requiring rapid reconstruction from dense inputs.
- **Limitations:** Dependent on high-quality dense initialization; may not generalize well to sparse input scenarios where standard 300,000 iterations are needed for convergence.

## Background
This training schedule is associated with methods like GBR (Generative Bundle Refinement) which combine dense reconstruction backbones (e.g., DUSt3R) with Gaussian Splatting [S1]. The approach exploits the fact that dense initialization reduces the search space, allowing convergence in far fewer steps [S1].

## Sources
[S1] GBR: Generative Bundle Refinement for High-Fidelity Gaussian Splatting With Enhanced Mesh Reconstruction, https://doi.org/10.1109/tcsvt.2025.3596613, fetched 2026-10-29
