# Differentiable Sphere Tracing

Also called: Differentiable Sphere Tracing Renderer, DIST

## Overview
Differentiable Sphere Tracing (DIST) is a rendering algorithm for implicit Signed Distance Functions (SDFs) represented by neural networks, enabling efficient forward and backward passes for inverse graphics optimization [S1]. It solves the non-differentiability and computational prohibitive nature of naive sphere tracing in deep learning frameworks by optimizing the rendering pipeline for GPU efficiency [S1]. It sets itself apart from standard sphere tracing by providing explicit gradient computation with respect to scene parameters (e.g., latent codes, camera poses) and from [Differentiable Silhouette Rendering](4322-differentiable-silhouette-rendering.md) by supporting full volumetric/photometric accumulation rather than just binary masks.

## How it works
**Inputs:**
*   Camera parameters: intrinsic matrix $K$, extrinsic rotation $R$ and translation $t$ (or camera position $c$ and direction vectors $\tilde{v}$).
*   Neural SDF network $f_\theta: \mathbb{R}^3 \to \mathbb{R}$ (or grid-based SDF).
*   Rendering parameters: `march_step` (iterations), `convergence_threshold` (epsilon), `bounding_radius` (optional).

**Outputs:**
*   Rendered image (depth, color, or silhouette).
*   Gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for backpropagation.

**Data Structures:**
*   `ray_positions`: Tensor of shape $(H, W, 3)$ storing current 3D coordinates of rays.
*   `ray_directions`: Tensor of shape $(H, W, 3)$ storing normalized ray vectors.
*   `foreground_masks`: Boolean tensor of shape $(H, W, 1)$ indicating active rays.
*   `signed_distances`: Tensor of shape $(H, W, 1)$ storing $f(p)$ values.

**Steps:**

1.  **Ray Initialization:**
    Generate a 2D homogeneous grid for image dimensions $(H, W)$. Project to camera space using $K^{-1}$ to get calibrated coordinates. Apply extrinsic rotation $R^T$ and normalize to get `ray_directions`. Initialize `ray_positions` to camera center $c$ [S2, unofficial].
    If a `bounding_radius` is provided, compute intersection with the bounding sphere:
    $$ a = \|\tilde{v}\|^2, \quad b = 2 \tilde{v} \cdot c, \quad c_{sphere} = \|c\|^2 - r^2 $$
    $$ \Delta = b^2 - 4ac_{sphere} $$
    $$ t = \frac{-b - \sqrt{\Delta}}{2a} $$
    Update positions if hit: `ray_positions` = $c + t \tilde{v}$ where $\Delta \ge 0$ [S2, unofficial].

2.  **Forward Marching Loop:**
    For $n = 0$ to `march_step` - 1:
    a.  **Query SDF:** Compute `signed_distances` = $f(\text{ray\_positions})$ [S1].
    b.  **Convergence Check:** Determine `converged` = $\| \text{signed\_distances} \| < \text{convergence\_threshold}$ [S3, unofficial].
    c.  **Update Position:**
        If $n=0$:
        $$ \text{ray\_positions} \leftarrow \text{ray\_positions} + \text{ray\_directions} \cdot \text{signed\_distances} $$
        If $n>0$:
        $$ \text{ray\_positions} \leftarrow \text{ray\_positions} + \text{ray\_directions} \cdot \text{signed\_distances} \cdot \text{foreground\_masks} \cdot \neg \text{converged} $$
        [S3, unofficial].
    d.  **Mask Update:**
        If `bounding_radius` is set:
        $$ \text{bounded} = \| \text{ray\_positions} \| < \text{bounding\_radius} $$
        $$ \text{foreground\_masks} \leftarrow \text{foreground\_masks} \land \text{bounded} $$
        [S3, unofficial].
    e.  **Early Exit:** If $\forall \text{pixels}, \neg \text{foreground\_masks} \lor \text{converged}$, break loop [S3, unofficial].

3.  **Backward Pass (Gradient Approximation):**
    To avoid storing the full computational graph for every SDF query, detach `ray_positions` and enable gradients:
    $$ p^* = \text{ray\_positions.detach}(); \quad p^*.requires\_grad = \text{True} $$
    Re-evaluate SDF at $p^*$ to get `signed_distances_star` [S3, unofficial].
    Compute gradient of SDF w.r.t positions:
    $$ g_p = \nabla_p f(p^*) = \frac{\partial \text{signed\_distances\_star}}{\partial p^*} $$
    [S3, unofficial].
    Compute output gradient approximation:
    $$ g_{out} = -\frac{g_{out\_dot\_dir}}{g_{p\_dot\_dir}} $$
    Where $g_{out\_dot\_dir} = \sum g_{out} \cdot \tilde{v}$ and $g_{p\_dot\_dir} = \sum g_p \cdot \tilde{v}$ [S3, unofficial].
    Zero gradients for unconverged points:
    $$ g_{out} \leftarrow g_{out} \cdot \text{converged} $$
    [S3, unofficial].
    Compute gradients w.r.t SDF parameters $\theta$:
    $$ \frac{\partial \mathcal{L}}{\partial \theta} = \frac{\partial \text{signed\_distances\_star}}{\partial \theta} \cdot g_{out} $$
    [S3, unofficial].

**Parameters:**
*   `convergence_threshold`: Default $5 \times 10^{-5}$ [S1]. Large values dilate shape; small values may produce incomplete geometry [S1].
*   `march_step`: Default 50 [S2, unofficial].
*   `bounding_radius`: Optional. Clamps rays to spherical volume [S2, unofficial].

**Pitfalls:**
*   Naive sphere tracing is prohibitive for neural networks due to recursive computational graphs [S1].
*   Division by zero in gradient calculation: clamp `grad_positions_dot_directions` to $[-10^{-6}, 10^{-6}]$ [S3, unofficial].
*   Non-intersecting pixels must be handled explicitly (e.g., mask update) to prevent artifacts [S4, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_sphere_trace(camera_pos, ray_dirs, sdf_network, 
                                march_step=50, threshold=5e-5, 
                                bounding_radius=None):
    """
    Forward and backward pass for differentiable sphere tracing.
    """
    # 1. Initialize
    ray_pos = camera_pos.unsqueeze(0).unsqueeze(0) # (1, 1, 3) -> (H, W, 3) broadcasted
    # Assuming ray_dirs is (H, W, 3)
    foreground_masks = torch.ones_like(ray_dirs[..., 0:1], dtype=torch.bool)
    
    # 2. Forward Marching
    for i in range(march_step):
        # Query SDF
        signed_dists = sdf_network(ray_pos) # (H, W, 1)
        
        # Convergence
        converged = torch.abs(signed_dists) < threshold
        
        # Update position
        step = ray_dirs * signed_dists
        if i == 0:
            ray_pos = ray_pos + step
        else:
            ray_pos = ray_pos + step * foreground_masks.float() * (~converged).float()
            
        # Update masks
        if bounding_radius is not None:
            dist_from_center = torch.norm(ray_pos, dim=-1, keepdim=True)
            bounded = dist_from_center < bounding_radius
            foreground_masks = foreground_masks & bounded
            
        if not torch.any(foreground_masks & ~converged):
            break
            
    # 3. Backward Pass (Approximation)
    # Detach and re-enable grad
    ray_pos_detached = ray_pos.detach()
    ray_pos_detached.requires_grad_(True)
    
    # Re-evaluate SDF for gradient computation
    signed_dists_detached = sdf_network(ray_pos_detached)
    
    # Compute gradient w.r.t positions
    grad_outputs = torch.ones_like(signed_dists_detached)
    grad_p, = torch.autograd.grad(
        outputs=signed_dists_detached, 
        inputs=ray_pos_detached, 
        grad_outputs=grad_outputs, 
        retain_graph=True, 
        create_graph=True
    )
    
    # Dot products
    grad_p_dot_dir = torch.sum(grad_p * ray_dirs, dim=-1, keepdim=True)
    # Note: grad_outputs here represents the upstream gradient from the loss
    # In a real implementation, this would be passed in. 
    # Here we assume we are computing the gradient of the SDF value itself 
    # to propagate to parameters.
    
    # Avoid division by zero
    grad_p_dot_dir = torch.clamp(grad_p_dot_dir, min=-1e-6, max=1e-6)
    
    # Approximate output gradient
    # If we want d(L)/d_params, we need d(L)/d(SDF) * d(SDF)/d_params
    # The term d(SDF)/d_pos is grad_p.
    # The chain rule for ray marching: d(L)/d_pos = - d(L)/d(SDF) * (d(SDF)/d_pos . v) / (d(SDF)/d_pos . v) ?
    # The paper uses: grad_outputs = - (grad_outputs . v) / (grad_p . v)
    # This assumes the loss depends on the hit position, which depends on SDF.
    
    # Zero out unconverged
    # grad_outputs_masked = grad_outputs * converged
    
    # Compute gradients w.r.t SDF parameters
    # This requires the upstream gradient from the loss function.
    # For this self-contained example, we return the SDF gradients w.r.t positions.
    return ray_pos, grad_p
```

## Subtopics

### [Instant-NSR-PL Forward Pass](2540-instant-nsr-pl-forward-pass.md)
The Instant-NSR-PL Forward Pass is the core rendering kernel for the Instant-Neural SDF Rendering (Instant-NSR) method, which renders scenes using differentiable sphere tracing. It computes the composite RGB image, surface normals, depth, and opacity by marching along rays, querying a neural Signed Distance Function (SDF) network, and accumulating color based on opacity derived from the SDF value and surface orientation. It is distinguished from its sibling [Differentiable Silhouette Rendering](4322-differentiable-silhouette-rendering.md) by its volumetric accumulation of color and normals rather than binary silhouette masks, and from the [SDF Direct Diffuse Integrator](4467-sdf-direct-diffuse-integrator.md) by its use of a learned texture network and finite-difference or analytical gradients for the SDF rather than direct analytical integration of diffuse shading.

### [Differentiable Silhouette Rendering](4322-differentiable-silhouette-rendering.md)
Differentiable Silhouette Rendering is a technique to render binary silhouettes of 3D shapes represented by Signed Distance Functions (SDFs) in a way that allows gradient back-propagation to scene parameters. It solves the non-differentiability of standard binary silhouette detection (e.g., ray termination at infinity) by computing a "soft" silhouette value based on the minimum absolute SDF encountered along a ray. It sets itself apart from its parent [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md) by focusing specifically on the silhouette supervision signal rather than full photometric or geometric rendering, and from siblings like [Instant-NSR-PL Forward Pass](2540-instant-nsr-pl-forward-pass.md) by relying on the minimum absolute distance query rather than normal-based edge detection.

### [SDF Direct Diffuse Integrator](4467-sdf-direct-diffuse-integrator.md)
The SDF Direct Diffuse Integrator is a differentiable volume rendering method that computes direct illumination for scenes defined by Signed Distance Functions (SDFs). It extends standard path tracing by incorporating a boundary integral term evaluated at grazing points, which enables accurate gradient computation for thin surfaces and silhouettes where standard sphere tracing fails. Unlike [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md), which relies on standard ray-marching intersections, this method explicitly handles the discontinuity of visibility at object boundaries using a relaxed boundary formulation.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the number of marching steps. The coarse-to-fine approach and aggressive marching strategy reduce the effective $M$ compared to naive tracing [S1].
*   **Memory Complexity:** $O(N)$ for storing ray positions and masks. The gradient approximation avoids storing the full computational graph, significantly reducing memory consumption compared to naive backpropagation [S1].
*   **Performance:** Efficient on commodity GPUs. The method enables inverse optimization (e.g., shape completion, camera pose estimation) that would otherwise be computationally prohibitive [S1].
*   **Strengths:** Fully differentiable, supports inverse graphics, handles complex implicit shapes via neural networks, efficient memory usage.
*   **Limitations:** Requires careful tuning of `convergence_threshold` to balance detail and completeness [S1]. Performance depends on the complexity of the neural SDF network.

## Background
Differentiable Sphere Tracing (DIST) was proposed by Liu et al. in 2020 [S1]. It bridges the gap between inverse graphics methods and deep learning-based implicit signed distance functions. The key innovation is optimizing both the forward and backward passes of the rendering layer to make it tractable for GPU-based optimization, addressing the prohibitive memory and computation costs of naive sphere tracing with neural networks [S1]. It has been applied to 3D shape prediction from single-view depth images and multi-view color images, outperforming methods like DeepSDF and PMO in certain scenarios [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S3] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S4] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
