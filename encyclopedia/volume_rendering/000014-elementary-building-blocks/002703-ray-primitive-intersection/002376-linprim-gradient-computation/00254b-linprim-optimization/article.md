# LinPrim Optimization

Also called: LinPrim Differentiable Optimization

## Overview
LinPrim Optimization is the parameter update phase of the LinPrim rendering pipeline, which adjusts the geometric and appearance features of linear primitives (triangles) to minimize a differentiable loss function [S1]. It differs from standard 3D Gaussian Splatting (3DGS) optimization by backpropagating gradients through a linear primitive rasterizer (MTIA) rather than a Gaussian splatting kernel, requiring specific handling of ray-space to world-space transformations and quaternion-based rotation updates [S1]. It builds on [LinPrim Gradient Computation](3038-linprim-gradient-computation.md) to compute the necessary gradients for intersection depths [S1].

## How it works
The optimization process minimizes a loss function using the ADAM optimizer, backpropagating gradients through the entire differentiable rendering pipeline [S1].

**Inputs:**
*   Current primitive parameters: position $\mathbf{p}$, quaternion $\mathbf{q}$, scale/distance features $\mathbf{s}$, view-dependent color coefficients, and opacity $\alpha$ [S1].
*   Training views: camera poses and intrinsics [G].
*   Target images: ground truth RGB images [G].
*   Loss weights: $\lambda_{\text{L1}}$ and $\lambda_{\text{SSIM}}$ following the 3DGS formulation [S1].

**Outputs:**
*   Updated primitive parameters for the next optimization step [S1].

**Data Structures:**
*   Primitives are stored as a list of objects containing:
    *   `center`: 3D position vector [S1].
    *   `quaternion`: 4D unit quaternion for rotation [S1].
    *   `distances`: 3D vector representing scale/extent features [S1].
    *   `color`: View-dependent color coefficients [S1].
    *   `opacity`: Scalar alpha value [S1].

**Steps:**

1.  **Forward Pass:** Render the scene using the LinPrim rasterizer to produce an image $I_{\text{rendered}}$ [S1].
2.  **Loss Computation:** Compute the total loss $\mathcal{L}$ as a weighted sum of L1 loss and SSIM loss between $I_{\text{rendered}}$ and the target image $I_{\text{target}}$, following the 3DGS formulation [S1].
3.  **Rasterization Backpropagation:**
    *   Compute gradients of the loss with respect to the output alpha $\alpha$ and color $C$ of each primitive [S1].
    *   The gradient $\frac{\partial \mathcal{L}}{\partial \alpha}$ indicates the desired transparency change [S1]. If $\alpha$ needs to increase, intersections should move closer; if decrease, farther apart [S1].
    *   Propagate gradients from $\alpha$ to the intersection depths $d$ along the ray [S1].
    *   Compute gradients of intersection depths with respect to the triangle corner positions in ray space, $\frac{\partial d}{\partial \mathbf{v}^{\text{ray}}}$, using the results from [LinPrim Gradient Computation](3038-linprim-gradient-computation.md) [S1].
    *   Aggregate gradients over all corners and both intersections (entry and exit) to obtain gradients with respect to the ray-space center and corner offsets [S1].
4.  **Preprocessing Backpropagation (Ray Space to World Space):**
    *   Reverse the view and projection transformations to convert ray-space gradients to world-space gradients [S1].
    *   Compute the final gradient with respect to the primitive center $\mathbf{p}$ [S1]. This combines the propagated ray-space center gradients with the gradient arising from the impact of position on the ray-space approximation and view-dependent color [S1].
    *   Split the gradient flow with respect to the corners when undoing the rotation [S1]. The gradients are distributed to the quaternion $\mathbf{q}$ and the distance features $\mathbf{s}$ [S1].
5.  **Learning Rate Adjustment:**
    *   Calculate the maximum distance $D_{\text{max}}$ between any two training cameras to approximate scene size [S1].
    *   Adjust learning rates for position and distance features based on $D_{\text{max}}$ to handle scale ambiguity [S1].
6.  **Parameter Update:**
    *   Apply the ADAM optimizer to update $\mathbf{p}$, $\mathbf{q}$, $\mathbf{s}$, color, and opacity using the computed gradients [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_linprim_step(primitives, training_views, target_images, optimizer, lambda_l1, lambda_ssim):
    optimizer.zero_grad()
    total_loss = 0.0
    for view, target in zip(training_views, target_images):
        rendered_img = forward_rasterize(primitives, view)
        l1_loss = lambda_l1 * mean_absolute_error(rendered_img, target)
        ssim_loss = lambda_ssim * compute_ssim(rendered_img, target)
        total_loss += l1_loss + ssim_loss
    total_loss /= len(training_views)
    total_loss.backward()
    max_cam_dist = 0.0
    for i, v1 in enumerate(training_views):
        for v2 in training_views[i+1:]:
            d = vector_norm(v1.position - v2.position)
            if d > max_cam_dist: max_cam_dist = d
    # Scale gradients for position and distance features based on max_cam_dist
    # [G] The sources state learning rates are adjusted based on max_cam_dist, but do not specify the exact scaling formula.
    for prim in primitives:
        prim.grad_position /= max_cam_dist
        prim.grad_distances /= max_cam_dist
    optimizer.step()

def forward_rasterize(primitives, view):
    # [G] Standard differentiable rasterization forward pass
    return render_image(primitives, view)

def mean_absolute_error(img1, img2):
    return sum(abs(img1[i][j][k] - img2[i][j][k]) for i in range(H) for j in range(W) for k in range(3)) / (H * W * 3)

def compute_ssim(img1, img2):
    # [G] Standard SSIM computation
    return ssim_metric(img1, img2)

def vector_norm(v):
    return sqrt(sum(v[j]**2 for j in range(3)))
```

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** Scenes are scale-ambiguous, requiring careful adjustment of learning rates for position and distance features based on scene size [S1].
*   **Quaternion Normalization:** When updating quaternions, normalization must be applied to maintain unit length [G].
*   **Degenerate Triangles:** If primitives collapse to lines or points, gradient computation may become unstable [G].

## Subtopics

### LinPrim Rasterization
The forward pass that renders linear primitives. It computes intersection depths and blends colors. [LinPrim Rasterization](3037-linprim-rasterization.md)

### LinPrim Gradient Computation
Computes gradients of intersection depths with respect to ray space triangle corners. [LinPrim Gradient Computation](3038-linprim-gradient-computation.md)

### 3D Gaussian Splatting Optimization
The baseline optimization method using Gaussian primitives, which LinPrim adapts. [3D Gaussian Splatting Optimization](3036-3d-gaussian-splatting-optimization.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Empirically mirrors the behavior of 2D Mip filters, reducing visible aliasing [S1].
*   **Strengths:** Differentiable rendering allows end-to-end optimization of linear primitives [S1]. The use of L1 and SSIM losses provides robust convergence [G].
*   **Limitations:** Requires careful handling of scale ambiguity and quaternion updates [S1]. Population control (adding/removing primitives) is not part of this optimization step and requires separate mechanisms [S1].
*   **When to Use:** When optimizing linear primitives for differentiable volumetric rendering where aliasing reduction is important [G].
*   **When Not to Use:** When Gaussian primitives are sufficient or when computational resources are extremely limited, as linear primitives may have higher overhead [G].

## Background
LinPrim Optimization was introduced in the paper "LinPrim: Linear Primitives for Differentiable Volumetric Rendering" [S1]. It extends the differentiable rendering framework of 3D Gaussian Splatting to linear primitives (triangles) [S1]. The method uses the ADAM optimizer and backpropagates through the entire rendering pipeline, including a custom rasterizer (MTIA) and preprocessing steps [S1]. The approach addresses visible aliasing by incorporating filtering mechanisms that mimic 2D Mip filters [S1].

## Sources
[S1] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
