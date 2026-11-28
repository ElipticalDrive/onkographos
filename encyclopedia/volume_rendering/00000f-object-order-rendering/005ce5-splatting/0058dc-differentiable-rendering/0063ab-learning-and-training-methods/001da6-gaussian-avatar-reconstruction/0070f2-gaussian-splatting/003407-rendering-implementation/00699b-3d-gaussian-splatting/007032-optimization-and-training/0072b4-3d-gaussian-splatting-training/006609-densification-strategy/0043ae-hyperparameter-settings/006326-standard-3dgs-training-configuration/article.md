# Standard 3DGS Training Configuration

Also called: Standard 3DGS Training

## Overview
Standard 3DGS Training Configuration specifies the canonical hyperparameter settings for optimizing 3D Gaussian Splatting models, establishing the baseline for adaptive density control and gradient-based optimization [S1]. It differs from sibling methods like [Conditional 3DGS Optimization and Rendering](11459-conditional-3dgs-optimization-and-rendering.md) or [Blended Error Densification and Splitting](13892-blended-error-densification-and-splitting.md) by adhering strictly to the default parameters of the official implementation, specifically regarding densification frequency, gradient thresholds, and opacity management [S1].

## How it works
The method configures the training loop to balance reconstruction quality and computational efficiency through a fixed schedule of densification, pruning, and opacity resets [S1].

**Inputs and Outputs**
*   **Inputs:** Training images, an initial point cloud, and the hyperparameter configuration [S1].
*   **Outputs:** Optimized set of 3D Gaussians [S1].

**Data Structures**
The Gaussian representation maintains per-Gaussian attributes:
*   **Position:** 3D vector $\mathbf{x} \in \mathbb{R}^3$.
*   **Covariance:** 3D rotation $\mathbf{R}$ and scaling $\mathbf{s}$, stored as quaternions and scale vectors [G].
*   **Opacity:** Scalar $\alpha \in [0, 1]$.
*   **Color:** Spherical Harmonics (SH) coefficients [G].

**Parameters**
*   **Maximum Iterations:** 30,000 [S1].
*   **D-SSIM Loss Weight:** 0.2 [S1].
*   **Spherical Harmonics Degree:** 3 [S1].
*   **Background Color:** Black [S1].
*   **Densification Gradient Threshold:** 0.0002 [S1].
*   **Opacity Reset Interval:** 3000 iterations [S1].
*   **Densification Start Iteration:** 500 [S1].
*   **Densification Interval:** 100 iterations [S1].
*   **`densify_until_iter`:** Controls the adaptive density-control block (accumulation, densification/pruning, periodic opacity reset); defaults to 30,000 [S1].
*   **`percent_dense`:** Threshold (0 to 1, multiplied by scene extent) classifying Gaussians as small or large; default 0.01 [S2].
*   **`densify_min_opacity`:** Opacity threshold filtering Gaussians during densification; default 0.005 [S2].

**Steps**
1.  **Initialization:** Initialize Gaussians from Structure-from-Motion (SfM) points with initial opacity 0.1 [G].
2.  **Optimization Loop:** Iterate $t$ from 1 to 30,000 [S1].
3.  **Rendering:** Render the current Gaussian set to generate images [G].
4.  **Loss Calculation:** Compute loss combining RGB and D-SSIM components with D-SSIM weight 0.2 [S1].
5.  **Gradient Accumulation:** Accumulate gradients for Gaussian positions and attributes [G].
6.  **Densification Check:** If $t \ge 500$, $(t - 500) \mod 100 == 0$, and $t \le \text{densify\_until\_iter}$:
    *   **Accumulate Statistics:** Track accumulated gradient magnitudes for each Gaussian [S1].
    *   **Split/Clone:** For Gaussians with accumulated gradient magnitude exceeding 0.0002 [S1]: classify as large or small using `percent_dense` $\times$ scene extent [S2]. Large Gaussians split into two copies with 80% of the original size [S2]. Small Gaussians clone identically [S2].
    *   **Pruning:** Remove Gaussians with opacity below 0.005 [S2].
    *   **Reset Statistics:** Clear accumulated gradients to zero [S1].
7.  **Opacity Reset:** If $t \mod 3000 == 0$ and $t \le \text{densify\_until\_iter}$, reset all Gaussian opacities to 0.01 [S1].
8.  **Optimization Step:** Update Gaussian parameters using the accumulated gradients [G].
9.  **Termination:** Stop at iteration 30,000 [S1].

**Edge Cases and Pitfalls**
*   **Precision:** Gradient thresholds are fixed at $2 \times 10^{-4}$ [S1]; ensure floating-point precision is sufficient to distinguish significant gradients from noise [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def standard_3dgs_training(gaussians, cameras, max_iter=30000, densify_until_iter=30000):
    densify_start_iter = 500
    densify_interval = 100
    densify_grad_threshold = 0.0002
    opacity_reset_interval = 3000
    opacity_reset_value = 0.01
    d_ssim_weight = 0.2
    sh_degree = 3
    background_color = [0.0, 0.0, 0.0]
    densify_min_opacity = 0.005
    percent_dense = 0.01
    scene_extent = 1.0  # Normalized scene bounds [G]
    accumulated_gradients = [0.0] * len(gaussians)

    for t in range(1, max_iter + 1):
        rendered_images = render(gaussians, cameras, background_color)
        loss = compute_loss(rendered_images, cameras, d_ssim_weight)
        gradients = compute_gradients(loss, gaussians)

        for i, g in enumerate(gaussians):
            accumulated_gradients[i] += norm(gradients[i].position)

        if t >= densify_start_iter and (t - densify_start_iter) % densify_interval == 0 and t <= densify_until_iter:
            new_gaussians = []
            for i, g in enumerate(gaussians):
                if accumulated_gradients[i] > densify_grad_threshold:
                    if g.scale_norm() > percent_dense * scene_extent:
                        new_gaussians.extend(split_gaussian(g, factor=0.8))
                    else:
                        new_gaussians.append(clone_gaussian(g))
                else:
                    new_gaussians.append(g)
            gaussians = [g for g in new_gaussians if g.opacity > densify_min_opacity]
            accumulated_gradients = [0.0] * len(gaussians)

        if t % opacity_reset_interval == 0 and t <= densify_until_iter:
            for g in gaussians:
                g.opacity = opacity_reset_value

        optimize(gaussians, gradients)

    return gaussians

def render(gaussians, cameras, background_color):
    # Standard differentiable splatting forward pass [G]
    return [splat(g, cam, background_color) for g, cam in zip(gaussians, cameras)]

def compute_loss(rendered, cameras, d_ssim_weight):
    # L1 loss + D-SSIM loss scaled by weight [S1]
    l1 = sum((r - t).abs() for r, t in zip(rendered, cameras))
    dssim = d_ssim_loss(rendered, cameras)
    return l1 + d_ssim_weight * dssim

def compute_gradients(loss, gaussians):
    # Standard autograd accumulation [G]
    return [g.grad for g in gaussians]

def optimize(gaussians, gradients):
    # Adam optimizer step [G]
    for g, grad in zip(gaussians, gradients):
        g.update(grad)

def norm(vec):
    return (sum(c**2 for c in vec))**0.5

def split_gaussian(g, factor):
    # Creates two copies with scale multiplied by factor [S2]
    return [copy_with_scale(g, factor), copy_with_scale(g, factor)]

def clone_gaussian(g):
    # Returns an identical copy [S2]
    return copy.deepcopy(g)
```

## Subtopics

### Modified 3DGS Densification
A parameter-tuning strategy that reduces the number of generated Gaussians and minimizes sorting time during training [S2]. It differs from the standard configuration by increasing the densification interval to 1000 (vs 100), lowering the densification gradient threshold to $7 \times 10^{-5}$ (vs $2 \times 10^{-4}$), raising the minimum opacity threshold to 0.1 (vs 0.005), deactivating the opacity reset (interval $\infty$ vs 3000), and increasing the `percent_dense` threshold to 0.1 (vs 0.01) [S2]. This configuration avoids regular pruning and creates fewer Gaussian splats overall [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1]
*   **Memory Complexity:** Not given in the sources. [S1]
*   **Performance:** Computational records and training times are tracked across runs [S1]. Experiments use Google Colab A100/H100 GPUs [S1].
*   **Strengths:** Not given in the sources. [S1]
*   **Limitations:** Not given in the sources. [S1]
*   **When to Use:** Not given in the sources. [S1]
*   **When Not to Use:** Not given in the sources. [S1]

## Background
The standard 3DGS training configuration originates from the official implementation of 3D Gaussian Splatting [S1]. It was standardized to ensure reproducibility across experiments [S1].

## Sources
[S1] Practical Evaluation of Standard 3D Gaussian Splatting for Real-World Scene Reconstruction., https://doi.org/10.3390/s26185754, fetched 2026-10-26
[S2] Compact 3D Scene Representation via Self-Organizing Gaussian Grids, https://doi.org/10.1007/978-3-031-73013-9_2, fetched 2026-10-26
