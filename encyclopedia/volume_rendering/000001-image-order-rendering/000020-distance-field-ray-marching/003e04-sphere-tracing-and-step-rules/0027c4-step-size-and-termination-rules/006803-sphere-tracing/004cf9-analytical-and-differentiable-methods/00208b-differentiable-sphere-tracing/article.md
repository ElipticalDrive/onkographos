# Differentiable Sphere Tracing

Also called: Differentiable Sphere Tracing Renderer, DIST

## Overview
Differentiable Sphere Tracing (DIST) is a rendering algorithm for implicit Signed Distance Functions (SDFs) represented by neural networks, enabling efficient forward and backward passes for inverse graphics optimization [S1]. It solves the problem of non-differentiable or computationally prohibitive ray marching in neural implicit representations by optimizing the rendering pipeline for GPU efficiency [S1]. It sets itself apart from naive sphere tracing by using a coarse-to-fine approach and aggressive ray marching to save computation, and from standard differentiable renderers by providing a specific gradient approximation that reduces memory consumption while maintaining training performance [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A neural network $f$ representing the SDF $f: \mathbb{R}^3 \rightarrow \mathbb{R}$, camera parameters (intrinsic matrix $K$, extrinsic rotation $R$, position $c$), image dimensions $(H, W)$, and rendering parameters (max iterations, convergence threshold, bounding volume).
- **Outputs:** Rendered image (RGB), depth, surface normals, opacity, and gradients with respect to SDF parameters and camera pose [S1][S2, unofficial].

**Data Structures:**
- **Ray Positions:** Tensor of shape $(H, W, 3)$ storing the current 3D position of each ray [S1].
- **Ray Directions:** Tensor of shape $(H, W, 3)$ storing normalized ray vectors [S1].
- **Foreground Masks:** Boolean tensor of shape $(H, W, 1)$ indicating which pixels have not yet converged or exited the bounding volume [S1].
- **SDF Values:** Tensor of shape $(H, W, 1)$ storing the signed distance at current positions [S1].

**Parameters:**
- `num_iterations`: Maximum number of ray marching steps. Typical value: 2000 [S3, unofficial].
- `convergence_threshold`: Distance threshold below which a ray is considered to have hit the surface. Typical value: $10^{-3}$ [S3, unofficial].
- `bounding_radius`: Optional radius for spherical bounding volume. If provided, rays are clamped to this radius [S3, unofficial].
- `march_step`: Integer, number of marching steps per ray (in some implementations). Default: 50 [S2, unofficial].
- `buffer_size`: Integer, size of the marching buffer. Default: 5 [S2, unofficial].
- `ray_marching_ratio`: Float, ratio for ray marching bounds. Default: 1.5 [S2, unofficial].
- `max_sample_dist`: Float, maximum sampling distance. Default: 0.2 [S2, unofficial].
- `threshold`: Float, SDF threshold for surface detection. Default: $5 \times 10^{-5}$ [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    - Initialize iteration counter $n=0$, accumulated distance $d^{(0)}=0$, and current position $p^{(0)} = c$ (camera center) [S1].
    - Initialize intrinsic camera matrix $K$ and its inverse $K_{inv}$ [S2, unofficial].
    - Generate a 2D homogeneous grid map for image dimensions $(H, W)$ and project it to camera space using $K_{inv}$ to create `homo_calib` (shape $3, H \times W$) [S2, unofficial].
    - Compute camera rays by rotating the calibrated homogeneous coordinates with the rotation matrix $R$: `rays` = $R^T \times \text{homo\_calib}$, then normalize the vectors [S2, unofficial].
    - Initialize `foreground_masks` to True for all finite ray positions: `foreground_masks = torch.all(torch.isfinite(ray_positions), dim=-1, keepdim=True)` [S1].
    - If `bounding_radius` is set, compute intersection with sphere:
      $$a = \sum (\text{ray\_directions} \cdot \text{ray\_directions})$$
      $$b = 2 \sum (\text{ray\_directions} \cdot \text{ray\_positions})$$
      $$c = \sum (\text{ray\_positions} \cdot \text{ray\_positions}) - \text{bounding\_radius}^2$$
      $$d = b^2 - 4ac$$
      $$t = \frac{-b - \sqrt{d}}{2a}$$
      Update positions if hit: `ray_positions = torch.where(bounded, ray_positions + ray_directions * t, ray_positions)` where `bounded = d >= 0` [S1].

2.  **Ray Marching Loop:**
    - Loop `num_iterations` times:
      - Query the SDF value $b^{(n)} = f(p^{(n)})$ at the current position using the neural network [S1].
      - Check convergence: `converged = torch.abs(signed_distances) < convergence_threshold` [S3, unofficial].
      - Update ray position:
        - If iteration $i > 0$: `ray_positions = torch.where(foreground_masks & ~converged, ray_positions + ray_directions * signed_distances, ray_positions)` [S3, unofficial].
        - If $i == 0$: `ray_positions = torch.where(foreground_masks, ray_positions + ray_directions * signed_distances, ray_positions)` [S3, unofficial].
        - Note: The step size is the signed distance value itself [S1].
      - If `bounding_radius` is set, update foreground masks: `bounded = torch.norm(ray_positions, dim=-1, keepdim=True) < bounding_radius`; `foreground_masks = foreground_masks & bounded` [S3, unofficial].
      - Break loop if all rays are either out of foreground or converged: `if torch.all(~foreground_masks | converged): break` [S3, unofficial].

3.  **Gradient Computation (Backward Pass):**
    - Detach ray positions and enable grad: `ray_positions = ray_positions.detach(); ray_positions.requires_grad_(True)` [S3, unofficial].
    - Compute gradient of SDF w.r.t. positions: `grad_positions, = torch.autograd.grad(outputs=signed_distances, inputs=ray_positions, grad_outputs=torch.ones_like(signed_distances), retain_graph=True)` [S3, unofficial].
    - Compute dot products:
      $$g_{out\_dir} = \sum (\text{grad\_outputs} \cdot \text{ray\_directions})$$
      $$g_{pos\_dir} = \sum (\text{grad\_positions} \cdot \text{ray\_directions})$$
    - Compute output gradient: `grad_outputs = -g_out_dir / g_pos_dir` [S3, unofficial].
    - Zero gradients for unconverged points: `grad_outputs = torch.where(converged, grad_outputs, torch.zeros_like(grad_outputs))` [S3, unofficial].
    - Compute gradients w.r.t. SDF parameters: `grad_parameters = torch.autograd.grad(outputs=signed_distances, inputs=parameters, grad_outputs=grad_outputs, retain_graph=True)` [S3, unofficial].
    - Return gradients [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def differentiable_sphere_tracing(sdf_network, camera_intrinsics, camera_extrinsics, image_height, image_width, 
                                  num_iterations=2000, convergence_threshold=1e-3, bounding_radius=None):
    """
    Performs differentiable sphere tracing for a neural SDF.
    
    Args:
        sdf_network: Neural network representing the SDF.
        camera_intrinsics: 3x3 intrinsic matrix K.
        camera_extrinsics: 3x3 rotation matrix R and 3x1 translation vector t.
        image_height: H
        image_width: W
        num_iterations: Max marching steps.
        convergence_threshold: SDF threshold for convergence.
        bounding_radius: Optional spherical bounding radius.
        
    Returns:
        rendered_image, depth, normals, gradients
    """
    # 1. Initialize Camera Rays
    # Create homogeneous grid (3, H, W)
    y, x = torch.meshgrid(torch.arange(image_height), torch.arange(image_width), indexing='ij')
    homo_grid = torch.stack([x, y, torch.ones_like(x)], dim=0).float() # Shape: (3, H, W)
    
    # Project to camera space: K_inv @ homo_grid
    K_inv = torch.inverse(camera_intrinsics)
    homo_calib = torch.matmul(K_inv, homo_grid.view(3, -1)) # Shape: (3, H*W)
    
    # Rotate to world space: R^T @ homo_calib
    R = camera_extrinsics[:3, :3]
    rays = torch.matmul(R.t(), homo_calib) # Shape: (3, H*W)
    rays = rays / torch.norm(rays, dim=0, keepdim=True) # Normalize
    rays = rays.view(3, image_height, image_width).permute(1, 2, 0) # Shape: (H, W, 3)
    
    # Camera position
    cam_pos = -torch.matmul(R.t(), camera_extrinsics[:3, 3:4]).squeeze() # Shape: (3,)
    
    # Initialize Ray Positions
    ray_positions = cam_pos.unsqueeze(0).unsqueeze(0).repeat(image_height, image_width, 1) # Shape: (H, W, 3)
    ray_directions = rays
    
    # Initialize Masks
    foreground_masks = torch.ones((image_height, image_width, 1), dtype=torch.bool, device=ray_positions.device)
    
    # Handle Bounding Radius
    if bounding_radius is not None:
        # Compute intersection with sphere
        a = torch.sum(ray_directions * ray_directions, dim=-1, keepdim=True)
        b = 2 * torch.sum(ray_directions * ray_positions, dim=-1, keepdim=True)
        c = torch.sum(ray_positions * ray_positions, dim=-1, keepdim=True) - bounding_radius ** 2
        d = b ** 2 - 4 * a * c
        # Clamp d to avoid NaNs
        d = torch.clamp(d, min=0.0)
        t = (-b - torch.sqrt(d)) / (2 * a)
        bounded = d >= 0
        ray_positions = torch.where(bounded, ray_positions + ray_directions * t, ray_positions)
        foreground_masks = foreground_masks & bounded.unsqueeze(-1)

    # 2. Ray Marching Loop
    for i in range(num_iterations):
        # Query SDF
        # Reshape positions for network input if necessary (e.g., batch dim)
        pos_flat = ray_positions.view(-1, 3)
        sdf_values = sdf_network(pos_flat).view(image_height, image_width, 1)
        
        # Check Convergence
        converged = torch.abs(sdf_values) < convergence_threshold
        
        # Update Positions
        # Only march if foreground and not converged
        march_mask = foreground_masks.squeeze(-1) & ~converged.squeeze(-1)
        
        if i == 0:
            # First step: march all foreground rays
            update_mask = foreground_masks.squeeze(-1)
        else:
            update_mask = march_mask
            
        # Avoid division by zero or invalid steps
        step_size = torch.where(update_mask, sdf_values, torch.zeros_like(sdf_values))
        ray_positions = torch.where(update_mask.unsqueeze(-1), ray_positions + ray_directions * step_size, ray_positions)
        
        # Update Foreground Masks
        if bounding_radius is not None:
            dist_from_origin = torch.norm(ray_positions, dim=-1, keepdim=True)
            bounded = dist_from_origin < bounding_radius
            foreground_masks = foreground_masks & bounded
        
        # Check if all rays are done
        if torch.all(~foreground_masks.squeeze(-1) | converged.squeeze(-1)):
            break
            
    # 3. Backward Pass (Gradient Computation)
    # Detach positions to break the computational graph from the forward pass
    ray_positions_detached = ray_positions.detach()
    ray_positions_detached.requires_grad_(True)
    
    # Re-query SDF for gradient computation
    pos_flat_detached = ray_positions_detached.view(-1, 3)
    sdf_values_detached = sdf_network(pos_flat_detached).view(image_height, image_width, 1)
    
    # Compute gradient of SDF w.r.t. positions
    grad_outputs = torch.ones_like(sdf_values_detached)
    grad_positions, = torch.autograd.grad(
        outputs=sdf_values_detached, 
        inputs=ray_positions_detached, 
        grad_outputs=grad_outputs, 
        retain_graph=True, 
        create_graph=True
    )
    
    # Compute dot products for chain rule approximation
    grad_outputs_dot_directions = torch.sum(grad_outputs * ray_directions, dim=-1, keepdim=True)
    grad_positions_dot_directions = torch.sum(grad_positions * ray_directions, dim=-1, keepdim=True)
    
    # Avoid division by zero
    grad_positions_dot_directions = torch.clamp(grad_positions_dot_directions, min=1e-6, max=1e6)
    
    # Compute output gradient
    grad_outputs = -grad_outputs_dot_directions / grad_positions_dot_directions
    
    # Zero gradients for unconverged points
    grad_outputs = torch.where(converged, grad_outputs, torch.zeros_like(grad_outputs))
    
    # Compute gradients w.r.t. SDF parameters
    # Note: In practice, this is done by calling backward() on the loss or using torch.autograd.grad on the network parameters
    # Here we simulate the gradient flow to the network weights
    # The actual implementation would pass grad_outputs back to the network
    # For the purpose of this pseudocode, we assume the network parameters are accessible
    # and we compute their gradients.
    
    # Return results
    # Depth is the accumulated distance along the ray
    # Depth = ||ray_positions - cam_pos||
    depth = torch.norm(ray_positions - cam_pos.unsqueeze(0).unsqueeze(0), dim=-1)
    
    # Normals are the normalized gradient of the SDF
    normals = grad_positions / (torch.norm(grad_positions, dim=-1, keepdim=True) + 1e-12)
    
    return ray_positions, sdf_values, normals, depth, grad_outputs
```

## Subtopics

### [Instant-NSR-PL Forward Pass](2540-instant-nsr-pl-forward-pass.md)
The Instant-NSR-PL Forward Pass renders scenes by marching rays through a neural Signed Distance Function (SDF) and accumulating color, normals, depth, and opacity along each ray. It computes the composite RGB image, surface normals, depth, and opacity by querying a geometry network for SDF values and gradients, computing opacity from the SDF and ray direction, and accumulating results along the ray.

### [Differentiable Silhouette Rendering](4322-differentiable-silhouette-rendering.md)
Differentiable Silhouette Rendering is a technique to render binary silhouettes of 3D shapes represented by Signed Distance Functions (SDFs) in a way that allows gradient back-propagation to scene parameters. It solves the non-differentiability of standard binary silhouette detection (e.g., ray termination at infinity) by computing a "soft" silhouette value based on the minimum absolute SDF encountered along a ray. It sets itself apart from its parent [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md) by focusing specifically on the silhouette supervision signal rather than full photometric or geometric rendering, and from siblings like [Instant-NSR-PL Forward Pass](2540-instant-nsr-pl-forward-pass.md) by relying on the minimum absolute distance query rather than normal-based edge detection.

### [SDF Direct Diffuse Integrator](4467-sdf-direct-diffuse-integrator.md)
The SDF Direct Diffuse Integrator is a differentiable rendering method that computes direct illumination for scenes defined by Signed Distance Functions (SDFs). It extends standard path tracing by incorporating a boundary integral term evaluated at grazing points, enabling accurate gradient computation for thin surfaces and silhouettes. Unlike standard differentiable sphere tracing, it explicitly handles visibility discontinuities at object boundaries using a relaxed boundary formulation.

### [Single-View Depth Shape Completion](5426-single-view-depth-shape-completion.md)
Recovers a complete 3D shape from a single depth image by optimizing a latent code through a differentiable renderer, requiring no additional training of the shape generator. It sets itself apart from its sibling [DeepSDF](link) by replacing fixed-distance sampling with adaptive sampling via differentiable sphere tracing, which better captures boundaries and thin structures. It differs from [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md) by explicitly combining a Directional Distance Function (DDF) for ray intersection with a Signed Distance Function (SDF) for loss computation, avoiding standard sphere tracing iterations.

### [Multi-View Photometric Shape Prediction](5427-multi-view-photometric-shape-prediction.md)
Predicts 3D shape from multiple color images by minimizing cross-view photometric consistency loss without requiring segmentation masks. It differs from encoder-based methods like PMO by using random initialization and an implicit representation, making it robust to initialization noise and camera focal length changes.

### [Camera Pose Optimization via Depth and Silhouette](5667-camera-pose-optimization-via-depth-and-silhouette.md)
This method optimizes an initial camera pose toward a ground truth pose by minimizing gradients computed from 2D depth and binary silhouette observations using a differentiable SDF renderer. It is distinct from sibling methods like [Instant-NSR-PL Forward Pass](2540-instant-nsr-pl-forward-pass.md) which may optimize shape parameters, by focusing specifically on pose recovery given a fixed shape. It differs from [Differentiable Silhouette Rendering](4322-differentiable-silhouette-rendering.md) by explicitly combining silhouette cross-entropy with a scale-invariant depth loss.

### Reparameterization for Differentiable SDF Rendering
A method using a reparameterization function $T$ for Signed Distance Functions (SDFs) that strictly satisfies the boundary requirement $\partial_\pi T(\omega_b, \pi) = \partial_\pi \omega_b$ [S5, unofficial]. It constructs a 3D vector field $\mathbf{V}$: $\mathbf{V}(x, \pi) := -\frac{\partial_x \phi(x, \pi_0)}{\phi(x, \pi) \|\partial_x \phi(x, \pi_0)\|_2}$, where $x \in \mathbb{R}^3$ is any point, and $\pi_0$ is the value of $\pi$ in the current scene (detached version) [S5, unofficial]. The evaluation distance function $t$ is constructed by reusing samples from sphere tracing: $t := \frac{\sum_{i=1}^N w^{(i)} t_i}{\sum_{i=1}^N w^{(i)}}$, where $t_i$ are intermediate distances and $w^{(i)}$ is a weighting function [S5, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity is proportional to the number of pixels, the number of iterations, and the cost of the neural network forward pass per iteration [S1].
- Memory complexity is dominated by storing the ray positions, directions, and intermediate SDF values for all pixels, as well as the computational graph for back-propagation [S1].

**Measured Performance:**
- DIST is designed to run efficiently on commodity GPUs with affordable memory consumption [S1].
- It uses a coarse-to-fine approach and aggressive ray marching to save computation compared to naive sphere tracing [S1].

**Strengths and Limitations:**
- **Strengths:** Fully differentiable, supports inverse graphics optimization, handles complex geometry via neural SDFs, efficient on GPUs [S1].
- **Limitations:** Prohibitive for naive sphere tracing due to recursive computational graphs; requires careful tuning of convergence thresholds and bounding volumes [S1]. Division by zero in gradient calculation must be handled by clamping [S3, unofficial].

**When to Use:**
- Use for inverse graphics problems involving neural implicit representations, such as 3D shape prediction from 2D images [S1].
- Use when differentiable rendering of SDFs is required for optimization [S2, unofficial].

**When Not to Use:**
- Do not use for explicit mesh rendering or voxel-based rendering where other methods are more efficient [S1].
- Do not use if high precision is required for very thin structures without adaptive sampling [S1].

## Background
Differentiable Sphere Tracing (DIST) was proposed by Liu et al. in 2020 [S1]. It bridges the gap between inverse graphics methods and deep learning-based implicit signed distance functions [S1]. The method optimizes both the forward and backward passes of the rendering layer to make it run efficiently with affordable memory consumption on a commodity graphics card [S1]. It enables geometric reasoning with strong generalization capability, allowing high-quality 3D shape prediction by performing iterative optimization with various 2D supervisions [S1]. The method has been implemented in PyTorch and used for single-view and multi-view inverse optimization [S2, unofficial].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S3] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S4] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
[S5] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
