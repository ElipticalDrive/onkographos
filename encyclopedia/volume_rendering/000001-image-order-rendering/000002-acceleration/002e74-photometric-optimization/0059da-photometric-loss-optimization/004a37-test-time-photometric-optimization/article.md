# Test-Time Photometric Optimization

Also called: Evaluation pose optimization, Gradient-driven camera pose refinement at test time, Test-time pose alignment via photometric error

## Overview
Test-Time Photometric Optimization is a post-processing refinement step applied to trained NeRF models to isolate view synthesis quality from camera pose estimation errors [S1]. It compensates for pose inaccuracies by optimizing evaluation camera poses during the validation phase, ensuring that quality metrics reflect rendering fidelity rather than alignment artifacts [S2, S1]. Unlike its siblings, which may optimize scene parameters or use specific geometric constraints, this method focuses exclusively on fine-grained pose alignment using photometric gradients while keeping the NeRF weights fixed [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A pre-trained NeRF model $\theta_{fixed}$, a set of test views with initial noisy camera poses $v_{init}$, and the corresponding target images $\hat{C}$ [S1, S2].
- **Output:** Refined camera poses $v_{opt}$ and the rendered images using these poses [S1].

**Data Structures:**
- Camera pose $v$: Represented as a transformation matrix or a vector of parameters defining rotation and translation [G].
- NeRF Model $\theta_{fixed}$: The trained radiance field parameters, frozen during this phase [S1].

**Steps:**
1. **Initialization:** Start with the initial noisy test poses $v_0 = v_{init}$ [S1].
2. **Ray Generation:** For each pixel in the target test image, generate a ray $r$ using the current camera pose $v_t$ [G].
3. **Differentiable Rendering:** Render the color $I(r; \theta_{fixed})$ for each ray using the fixed NeRF model [Parent].
4. **Loss Computation:** Compute the photometric loss $\mathcal{L}_{photo}$ between the rendered color and the target color $\hat{C}(r)$ [Parent]. The sources specify using only the RGB loss ($L_{rgb}$) for this optimization [S2].
   $$
   \mathcal{L}_{photo} = \frac{1}{|R|} \sum_{r \in R} \| I(r) - \hat{C}(r) \|^2
   $$
5. **Gradient Computation:** Compute the gradient of the loss with respect to the camera pose parameters $\nabla_{v} \mathcal{L}_{photo}$ via backpropagation through the differentiable renderer [Parent].
6. **Pose Update:** Update the camera pose using gradient descent:
   $$
   v_{t+1} = v_t - \eta \nabla_{v} \mathcal{L}_{photo}
   $$
   where $\eta$ is the learning rate [Parent].
7. **Iteration:** Repeat steps 3–6 until convergence or a maximum number of iterations is reached [G].
8. **Transformation:** Transform the test views to the coordinate system of the optimized poses by applying the scale/rotation/translation from the alignment analysis [S1].

**Parameters:**
- Learning rate $\eta$: Not given in the sources.
- Number of iterations: Not given in the sources.
- Loss type: $L_{rgb}$ only [S2].

**Edge Cases and Pitfalls:**
- **Occlusions:** High-confidence regions should be used to avoid artifacts from NeRF inaccuracies or occlusions, often utilizing masks or viewshed fields [Parent].
- **Precision:** Gradient-based optimization on pose parameters requires careful handling of rotation parameterizations to avoid singularities [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_time_photometric_optimization(nerf_model, test_views, target_images, learning_rate, max_iterations):
    """
    Optimizes camera poses for test views using photometric loss.
    
    Args:
        nerf_model: Pre-trained NeRF model (frozen).
        test_views: List of initial camera poses (v_init).
        target_images: List of target images for each test view.
        learning_rate: Step size for gradient descent.
        max_iterations: Maximum number of optimization steps.
        
    Returns:
        optimized_poses: List of refined camera poses.
    """
    optimized_poses = [v.clone() for v in test_views]
    
    for i in range(max_iterations):
        total_loss = 0.0
        pose_gradients = []
        
        for v, target_img in zip(optimized_poses, target_images):
            # Generate rays from current pose
            rays = generate_rays_from_pose(v)
            
            # Render image using fixed NeRF
            rendered_img = nerf_model.render(rays)
            
            # Compute L_rgb loss (MSE)
            loss = mean_squared_error(rendered_img, target_img)
            
            # Backpropagate to get pose gradients
            loss.backward()
            grad_v = v.grad.clone()
            pose_gradients.append(grad_v)
            
            total_loss += loss.item()
            
        # Update poses
        for j, v in enumerate(optimized_poses):
            v -= learning_rate * pose_gradients[j]
            
    return optimized_poses

def generate_rays_from_pose(pose):
    # Standard pinhole camera projection for each pixel in the image grid
    return [project_pixel_to_ray(pose, pixel) for pixel in image_grid]

def mean_squared_error(rendered, target):
    # Element-wise squared difference averaged over all pixels
    return sum((r - t) ** 2 for r, t in zip(rendered, target)) / len(rendered)
```

## Subtopics

### [Inverse Manipulation (Inversion)](16780-inverse-manipulation-inversion.md)
Inverse Manipulation is an iterative optimization method that inversely projects an input image $I_r$ onto the generation manifold of a pre-trained conditional NeRF by optimizing the generation conditions: shape code $z_s$, appearance code $z_a$, and camera pose $v$. It distinguishes itself from direct latent code retrieval by using an Expectation-Maximization (EM) style alternating optimization to find the specific latent codes and camera parameters that best reconstruct the input image, enabling subsequent text-driven or attribute-based manipulations. Unlike [Test-Time Photometric Optimization](12699-test-time-photometric-optimization.md) which may optimize for general fidelity, this method specifically targets the disentangled latent space of a conditional NeRF to enable precise editing.

## Operating characteristics
- **Time Complexity:** Depends on the number of iterations and the cost of differentiable rendering per iteration [G]. Rendering a $1920 \times 1080$ image takes $\sim 16.4$ seconds in one reported implementation [S2].
- **Memory Complexity:** Requires storing the NeRF model and intermediate gradients for the camera poses [G].
- **Strengths:** Factors out pose error from view synthesis quality measurement, providing a fairer evaluation of rendering fidelity [S1].
- **Limitations:** Computationally expensive due to iterative optimization at test time [G].
- **When to Use:** When evaluating novel-view synthesis quality in the presence of noisy or drifting camera poses [S1, S2].
- **When Not to Use:** Not given in the sources.

## Background
Test-Time Photometric Optimization was introduced to address the issue of pose errors contaminating view synthesis quality measurements in NeRFs trained on sparse or noisy poses [S1]. It builds on the idea that optimizing evaluation poses can compensate for drifting geometry introduced by optimized training poses [S2]. The method is commonly used in experiments where poses are refined, and also in experiments with fixed initial noisy poses for fairness [S1]. It is often used in conjunction with metrics like PSNR, SSIM, and LPIPs to evaluate view-synthesis performance [S1].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S2] Multi-Level Neural Scene Graphs for Dynamic Urban Environments, https://doi.org/10.1109/cvpr52733.2024.01996, fetched 2026-10-29
