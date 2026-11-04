# Differentiable Volumetric Rendering (DVR)

Also called: Implicit Surface Differentiation, Implicit Surface Gradient via Implicit Differentiation, Implicit Surface Rendering

## Overview
Differentiable Volumetric Rendering (DVR) predicts pixel color by casting a ray from a camera origin through a pixel, finding the first intersection with an implicit isosurface defined by a neural network, and querying a texture network at that point [S1, S2][S3, unofficial]. It solves the problem of optimizing implicit neural representations of 3D shape and texture directly from 2D RGB images without requiring 3D ground truth supervision [S2][S3, unofficial]. Unlike voxel-based differentiable rendering methods or LSTM-based renderers, DVR does not store intermediate volumetric data or unrolled computation graphs for the forward pass, making its memory footprint independent of sampling accuracy and allowing for higher resolution [S1, S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera origin $\mathbf{r}_0$, ray direction vector $\mathbf{w}$ (from $\mathbf{r}_0$ to pixel $u$), network parameters $\theta$, threshold $\tau$, sampling count $n$, step size $\Delta s$, and start depth $s_0$ [S1, S2][S3, unofficial].
*   **Outputs:** Rendered color $\hat{\mathbf{I}}_u$ and gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ with respect to a loss $\mathcal{L}$ [S1, S2][S3, unofficial].

**Data Structures:**
*   **Occupancy Network $f_\theta$:** Maps $\mathbb{R}^3 \times \mathcal{Z} \rightarrow [0, 1]$ and outputs an occupancy probability [S2][S3, unofficial].
*   **Texture Network $t_\theta$:** Maps $\mathbb{R}^3 \times \mathcal{Z} \rightarrow \mathbb{R}^3$ and outputs an RGB color value [S2][S3, unofficial].
*   **Implementation:** $f_\theta$ and $t_\theta$ are typically implemented as a single neural network with two shallow output heads sharing hidden layers [S2][S3, unofficial].
*   **Latent Embedding $z$:** For single-view reconstruction, an encoder $g_\theta$ produces $z \in \mathcal{Z}$ conditioned on the input image, which modulates both heads [S2][S3, unofficial]. For unconditional rendering, $\mathcal{Z}$ is empty or fixed [G].

**Parameters:**
*   $\tau$: Isosurface threshold. Typical value: $0.5$ [S1, S2][S3, unofficial]. Range: $[0, 1]$ [S2][S3, unofficial].
*   $n$: Number of equally-spaced samples on the ray. Starts at $16$, increases to $128$ during training (doubling after 50k, 150k, 250k iterations) [S1, S2][S3, unofficial].
*   $s_0$: Start depth of the sampling interval [S1, S2][S3, unofficial].
*   $\Delta s$: Step size for samples [S1, S2][S3, unofficial].
*   $\mathbf{w}$: Ray direction vector $\mathbf{w} = \mathbf{u} - \mathbf{r}_0$ [S1, S2][S3, unofficial].

**Algorithm Steps:**

1.  **Ray Sampling:** Compute $n$ points along the ray $\mathbf{r}(d) = \mathbf{r}_0 + d\mathbf{w}$:
    $$ \mathbf{p}_j^{\text{ray}} = \mathbf{r}(j\Delta s + s_0) \quad \text{for } j = 1, \dots, n $$
    [S1, S2][S3, unofficial]

2.  **Intersection Detection:** Find the smallest index $j$ where the occupancy changes from free space to occupied space:
    $$ j = \operatorname{argmin}_{j'} \left( f_\theta(\mathbf{p}_{j'+1}^{\text{ray}}) \geq \tau > f_\theta(\mathbf{p}_{j'}^{\text{ray}}) \right) $$
    [S1, S2][S3, unofficial]
    *Edge Case:* If no such $j$ exists (ray never intersects or starts inside), the depth is undefined or handled by boundary conditions [G].

3.  **Depth Approximation:** Approximate the surface depth $\hat{d}$ using the iterative secant method on the interval $[j\Delta s + s_0, (j+1)\Delta s + s_0]$ [S1, S2][S3, unofficial]. Let $\hat{\mathbf{p}} = \mathbf{r}(\hat{d})$ be the intersection point [S1, S2][S3, unofficial].

4.  **Color Prediction:** Compute the rendered color:
    $$ \hat{\mathbf{I}}_u = t_\theta(\hat{\mathbf{p}}) $$
    [S1, S2][S3, unofficial]

5.  **Gradient Computation (Backward Pass):**
    The total loss gradient is:
    $$ \frac{\partial \mathcal{L}}{\partial \theta} = \sum_u \frac{\partial \mathcal{L}}{\partial \hat{\mathbf{I}}_u} \cdot \frac{\partial \hat{\mathbf{I}}_u}{\partial \theta} $$
    [S1, S2][S3, unofficial]
    
    Using the chain rule on $\hat{\mathbf{I}}_u = t_\theta(\hat{\mathbf{p}})$:
    $$ \frac{\partial \hat{\mathbf{I}}_u}{\partial \theta} = \frac{\partial t_\theta(\hat{\mathbf{p}})}{\partial \theta} + \frac{\partial t_\theta(\hat{\mathbf{p}})}{\partial \hat{\mathbf{p}}} \cdot \frac{\partial \hat{\mathbf{p}}}{\partial \theta} $$
    [S1, S2][S3, unofficial]
    
    Since $\hat{\mathbf{p}} = \mathbf{r}(\hat{d}) = \mathbf{r}_0 + \hat{d}\mathbf{w}$:
    $$ \frac{\partial \hat{\mathbf{p}}}{\partial \theta} = \mathbf{w} \frac{\partial \hat{d}}{\partial \theta} $$
    [S1, S2][S3, unofficial]
    
    The depth gradient $\frac{\partial \hat{d}}{\partial \theta}$ is derived via implicit differentiation of $f_\theta(\hat{\mathbf{p}}) = \tau$:
    $$ \frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \theta} + \frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \hat{\mathbf{p}}} \cdot \frac{\partial \hat{\mathbf{p}}}{\partial \theta} = 0 $$
    Substituting $\frac{\partial \hat{\mathbf{p}}}{\partial \theta}$:
    $$ \frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \theta} + \frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \hat{\mathbf{p}}} \cdot \mathbf{w} \frac{\partial \hat{d}}{\partial \theta} = 0 $$
    Solving for $\frac{\partial \hat{d}}{\partial \theta}$:
    $$ \frac{\partial \hat{d}}{\partial \theta} = - \left( \frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \hat{\mathbf{p}}} \cdot \mathbf{w} \right)^{-1} \frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \theta} $$
    [S1, S2][S3, unofficial]
    
    **Implementation Trick:** To compute the gradient efficiently using reverse-mode automatic differentiation without storing the forward pass of the secant method, rewrite the term $\lambda \frac{\partial \hat{d}}{\partial \theta}$ (where $\lambda = \frac{\partial \mathcal{L}}{\partial \hat{d}}$ is the incoming gradient from the loss) as:
    $$ \mu \frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \theta} \quad \text{with} \quad \mu = - \left( \frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \hat{\mathbf{p}}} \cdot \mathbf{w} \right)^{-1} \lambda $$
    [S1, S2][S3, unofficial]
    
    The backward pass is implemented by:
    1.  Computing $\mu$ from the incoming gradient $\lambda$, the gradient of the occupancy network w.r.t. position $\frac{\partial f_\theta(\hat{\mathbf{p}})}{\partial \hat{\mathbf{p}}}$, and the ray direction $\mathbf{w}$ [S1, S2][S3, unofficial].
    2.  Element-wise multiplying $\lambda$ by $\mu$ (broadcasting for batches) [S1, S2][S3, unofficial].
    3.  Backpropagating the result through the occupancy network $f_\theta(\hat{\mathbf{p}})$ to get gradients w.r.t. $\theta$ [S1, S2][S3, unofficial].
    4.  Adding the direct gradient from the texture network $t_\theta(\hat{\mathbf{p}})$ w.r.t. $\theta$ and the term $\frac{\partial t_\theta(\hat{\mathbf{p}})}{\partial \hat{\mathbf{p}}} \cdot \mathbf{w} \frac{\partial \hat{d}}{\partial \theta}$ (which can also be computed via the same $\mu$ trick or standard AD if the texture network is differentiated through the depth) [S1, S2][S3, unofficial].
    5.  The texture gradient $\frac{\partial t_\theta(\hat{\mathbf{p}})}{\partial \theta}$ is standard [S1, S2][S3, unofficial].
    6.  The term $\frac{\partial t_\theta(\hat{\mathbf{p}})}{\partial \hat{\mathbf{p}}} \cdot \mathbf{w} \frac{\partial \hat{d}}{\partial \theta}$ requires $\frac{\partial \hat{d}}{\partial \theta}$, which can be computed using the $\mu$ formula derived from $f_\theta$ [S1, S2][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def dvr_forward(f_theta, t_theta, r0, w, tau, n, ds, s0):
    """
    Forward pass for DVR.
    f_theta: Occupancy network (returns scalar occupancy prob)
    t_theta: Texture network (returns RGB color)
    r0: Camera origin (3,)
    w: Ray direction (3,)
    tau: Threshold
    n: Number of samples
    ds: Step size
    s0: Start depth
    """
    # 1. Sample points on ray
    depths = torch.arange(1, n + 1, device=r0.device) * ds + s0
    points = r0 + depths[:, None] * w  # Shape: (n, 3)
    
    # 2. Evaluate occupancy
    occupancies = f_theta(points)  # Shape: (n,)
    
    # 3. Find intersection: first i where occ[i+1] >= tau > occ[i]
    free = occupancies < tau
    occupied_next = torch.roll(occupancies >= tau, -1)
    occupied_next[-1] = False
    intersect = occupied_next & free
    
    if not intersect.any():
        return None, None, None, None, None
        
    i = torch.argmax(intersect.float())
    
    # 4. Secant method on interval [d_i, d_{i+1}]
    d0 = i * ds + s0
    d1 = (i + 1) * ds + s0
    f0 = occupancies[i]
    f1 = occupancies[i+1]
    
    d_hat = d0
    for _ in range(10):  # [G] Fixed iterations for simplicity, source says "iterative"
        f_hat = f0 + (d_hat - d0) * (f1 - f0) / (d1 - d0)
        if abs(f_hat - tau) < 1e-6:
            break
        if f_hat < tau:
            d0, f0 = d_hat, f_hat
        else:
            d1, f1 = d_hat, f_hat
        d_hat = d0 + (tau - f0) * (d1 - d0) / (f1 - f0)
        
    # 5. Compute intersection point and color
    p_hat = r0 + d_hat * w
    color = t_theta(p_hat)
    
    return d_hat, color, p_hat, occupancies, intersect

def dvr_backward(loss_grad, d_hat, p_hat, f_theta, t_theta, w):
    """
    Backward pass for DVR using the implicit differentiation trick.
    loss_grad: Gradient of loss w.r.t. color (same shape as color)
    """
    # 1. Compute lambda = dL/dd_hat
    p_hat.requires_grad_(True)
    color = t_theta(p_hat)
    color.backward(loss_grad, retain_graph=True)
    d_color_dp = p_hat.grad  # Shape: (3,)
    lambda_val = torch.dot(d_color_dp, w)  # Scalar
    
    # 2. Compute mu = - (df/dp * w)^-1 * lambda
    f_out = f_theta(p_hat)
    f_out.backward(torch.ones_like(f_out), retain_graph=True)
    d_f_dp = p_hat.grad  # Shape: (3,)
    mu = - (torch.dot(d_f_dp, w)).reciprocal() * lambda_val
    
    # 3. Gradient through occupancy network (the "trick")
    f_theta.zero_grad()
    f_out2 = f_theta(p_hat)
    f_out2.backward(mu)
    grad_f_theta = f_theta.grad
    
    # 4. Direct gradient through texture network (detach p_hat to ignore depth dependency)
    p_hat_detached = p_hat.detach()
    p_hat_detached.requires_grad_(True)
    color_detached = t_theta(p_hat_detached)
    color_detached.backward(loss_grad, retain_graph=True)
    grad_t_direct = t_theta.grad
    
    # 5. Combine gradients
    total_grad = grad_t_direct + grad_f_theta
    
    return total_grad
```

## Subtopics

### [DVR Training Losses](809-dvr-training-losses.md)
A set of loss functions used to train the occupancy network $f_\theta$ and texture network $t_\theta$ based on whether a pixel $u$ is inside/outside the object mask and whether a finite surface depth is predicted.

### [DVR Network Architecture and Training Setup](850-dvr-network-architecture-and-training-setup.md)
Details on the encoder network, latent embedding, and training hyperparameters.

## Operating characteristics
*   **Time Complexity:** Dominated by the evaluation of the neural networks $f_\theta$ and $t_\theta$ at $n$ sample points per ray. $n$ increases during training from 16 to 128 [S1, S2][S3, unofficial].
*   **Memory Complexity:** Memory footprint is independent of the sampling accuracy $n$ and the volume resolution because no intermediate volumetric data is stored in the forward pass [S1, S2][S3, unofficial]. This contrasts with voxel-based methods which require storing all intermediate evaluations [S3, unofficial].
*   **Performance:** Demonstrated on single NVIDIA V100 GPU with batch size 64 images, 1024 random pixels each [S1, S2][S3, unofficial].
*   **Strengths:** Enables learning from 2D images without 3D supervision; high resolution due to no voxel grid; watertight meshes can be extracted [S2][S3, unofficial].
*   **Limitations:** Requires finding the first intersection; sensitive to the threshold $\tau$; secant method convergence [G].

## Background
DVR builds on occupancy networks and texture fields [S2][S3, unofficial]. It addresses the limitation of previous implicit representation methods that required 3D ground truth [S2][S3, unofficial]. It differs from voxel-based differentiable rendering and LSTM-based renderers by being inherently differentiable without storing intermediate results [S3, unofficial].

## Sources
[S1] Differentiable Volumetric Rendering: Learning Implicit 3D Representations without 3D Supervision, https://www.youtube.com/watch?v=lcub1KH-mmk, fetched 2026-10-27
[S2] Differentiable Volumetric Rendering: Learning Implicit 3D ..., https://www.cvlibs.net/publications/Niemeyer2020CVPR.pdf, fetched 2026-10-27
[S3] Differentiable Volumetric Rendering - CVF Open Access, https://openaccess.thecvf.com/content_CVPR_2020/html/Niemeyer_Differentiable_Volumetric_Rendering_Learning_Implicit_3D_Representations_Without_3D_Supervision_CVPR_2020_paper.html, fetched 2026-10-30 (unofficial)
