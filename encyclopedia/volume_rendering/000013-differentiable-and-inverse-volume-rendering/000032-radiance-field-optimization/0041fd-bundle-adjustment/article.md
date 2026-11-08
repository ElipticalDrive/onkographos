# Bundle Adjustment

Also called: Bundle Adjustment and Pose Optimization, Camera and Pose Optimization, Pose Optimization via Differentiable Rendering

## Overview
Bundle Adjustment (BA) in the context of differentiable volume rendering iteratively refines camera pose parameters (quaternion and translation) for a given view by minimizing the pixel-wise loss between the rendered image and the ground truth image using gradient descent [S1, unofficial]. It addresses the problem of inaccurate or missing camera poses, which are typically required as input for radiance field optimization [S1, unofficial]. Unlike methods that assume perfect pose initialization, BA treats camera extrinsics as learnable parameters, allowing the system to converge to optimal viewing parameters alongside the scene representation [S1, unofficial]. It is distinct from [Joint Optimization of NeRF and Camera Parameters](7492-joint-optimization-of-nerf-and-camera-parameters.md) in that it often operates as a refinement step on existing models or specific subsets of views, whereas joint optimization may initialize poses randomly [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained or partially trained scene representation (e.g., 3D Gaussians, NeRF), a set of training or test views with initial camera pose estimates (world-to-view transforms), and ground truth images for those views.
*   **Outputs:** Optimized camera pose tensors (quaternion and translation) for each view.

**Data Structures:**
*   **Camera Pose Tensor:** A tensor of shape `[N, 7]` for $N$ views, where the first 4 elements represent the quaternion $(q_w, q_x, q_y, q_z)$ and the last 3 elements represent the translation $(t_x, t_y, t_z)$.
*   **Optimizer State:** Adam optimizer state with specific parameter groups for translation and quaternion components.

**Numbered Steps:**
1.  **Initialization:** Extract the initial camera pose from the view's `world_view_transform`. Split this into a quaternion vector (first 4 elements) and a translation vector (last 3 elements). Enable gradient tracking (`requires_grad=True`) for both components.
2.  **Optimizer Setup:** Create an Adam optimizer with two distinct parameter groups:
    *   Translation parameters with learning rate $lr_t = 0.003$.
    *   Quaternion parameters with learning rate $lr_q = 0.001$.
    *   Shared hyperparameters: Betas $(\beta_1, \beta_2) = (0.9, 0.999)$, weight decay $= 10^{-4}$.
3.  **Scheduler Setup:** Create a Cosine Annealing Learning Rate scheduler with $T_{max} = \text{num\_iter}$ and minimum learning rate $\eta_{min} = 0.0001$.
4.  **Optimization Loop:** For each iteration $k$ from $0$ to $\text{num\_iter} - 1$:
    *   Concatenate the current quaternion and translation into a single `camera_pose` tensor.
    *   Render the scene using the differentiable renderer, passing the `camera_pose` to generate the predicted image $\hat{C}$.
    *   Compute a binary mask $M$ where $M_{ij} = 1$ if $\hat{C}_{ij} > \text{black\_hole\_threshold}$ (typically $0.0$), and $0$ otherwise. This masks out zero-valued pixels (e.g., background or empty space).
    *   Compute the loss $L$ as the masked L1 loss between the rendered image $\hat{C}$ and the ground truth image $C_{gt}$:
        $$
        L = \frac{\sum_{i,j} M_{ij} | \hat{C}_{ij} - C_{gt,ij} |}{\sum_{i,j} M_{ij}}
        $$
    *   Perform backpropagation: `loss.backward()`.
    *   Update parameters: `optimizer.step()`.
    *   Zero gradients: `optimizer.zero_grad(set_to_none=True)`.
    *   Update learning rate: `scheduler.step()`.
    *   **Tracking Best Pose:** If $k=0$, store $L_0$. If $L < L_{best}$, update $L_{best} = L$ and save the current `candidate_q` and `candidate_T`.
5.  **Finalization:** After the loop, assign the `candidate_q` and `candidate_T` (the pose that yielded the minimum loss) back to the main camera tensor. Render the final image using these optimized poses.

**Parameters:**
*   `num_iter`: Number of optimization iterations per view. (Not given in the sources, typically determined by command-line arguments).
*   `black_hole_threshold`: Threshold to mask out zero-valued pixels. Typical value: $0.0$ [S1, unofficial].
*   `lr_t`: Learning rate for translation. Value: $0.003$ [S1, unofficial].
*   `lr_q`: Learning rate for quaternion. Value: $0.001$ [S1, unofficial].
*   `eta_min`: Minimum learning rate for the scheduler. Value: $0.0001$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.optim.lr_scheduler import CosineAnnealingLR

def bundle_adjustment_pose_optimization(
    render_function,
    initial_world_view_transform,
    ground_truth_images,
    num_iter,
    black_hole_threshold=0.0
):
    """
    Optimizes camera poses (quaternion and translation) for a set of views.
    
    Args:
        render_function: A callable that takes (view_params, scene_params) and returns rendered image.
        initial_world_view_transform: Tensor of shape [N, 7] or [N, 4, 4] representing initial poses.
        ground_truth_images: Tensor of shape [N, H, W, 3] containing GT images.
        num_iter: Integer, number of optimization iterations.
        black_hole_threshold: Float, threshold for masking zero-valued pixels.
        
    Returns:
        optimized_poses: Tensor of shape [N, 7] with optimized quaternion and translation.
    """
    # Step 1: Initialization
    # Assuming initial_world_view_transform is [N, 7] [q_w, q_x, q_y, q_z, t_x, t_y, t_z]
    # If it is a 4x4 matrix, it must be decomposed first. Here we assume the [N, 7] format based on notes.
    pose_tensor = initial_world_view_transform.clone().detach().requires_grad_(True)
    
    # Split into quaternion and translation
    q = pose_tensor[:, :4].clone().detach().requires_grad_(True)
    t = pose_tensor[:, 3:].clone().detach().requires_grad_(True)
    
    # Step 2: Optimizer Setup
    # Two parameter groups: translation and quaternion
    optimizer = optim.Adam([
        {'params': [t], 'lr': 0.003},
        {'params': [q], 'lr': 0.001}
    ], betas=(0.9, 0.999), weight_decay=1e-4)
    
    # Step 3: Scheduler Setup
    scheduler = CosineAnnealingLR(optimizer, T_max=num_iter, eta_min=0.0001)
    
    # Tracking best loss and poses
    current_min_loss = float('inf')
    best_q = None
    best_t = None
    
    # Step 4: Optimization Loop
    for iteration in range(num_iter):
        # Concatenate q and t into camera_pose tensor
        # Note: The notes specify concatenating q and T. 
        # If q is [N, 4] and t is [N, 3], result is [N, 7]
        camera_pose = torch.cat([q, t], dim=1)
        
        # Render the scene
        # render_function expects the pose in the format used by the specific renderer
        # Assuming render_function takes the concatenated pose tensor
        rendered_image = render_function(camera_pose)
        
        # Compute mask: (rendering > 0.0).float()
        mask = (rendered_image > black_hole_threshold).float()
        
        # Compute loss: l1_loss_mask(rendering, gt, mask)
        # Masked L1 Loss
        masked_diff = mask * torch.abs(rendered_image - ground_truth_images)
        loss = torch.sum(masked_diff) / (torch.sum(mask) + 1e-8) # Add epsilon to avoid div by zero
        
        # Backpropagation
        loss.backward()
        
        # Optimizer step
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
        
        # Update scheduler
        scheduler.step()
        
        # Track best pose
        if iteration == 0:
            initial_loss = loss.item()
            
        if loss.item() < current_min_loss:
            current_min_loss = loss.item()
            # Clone tensors to save state
            best_q = q.detach().clone()
            best_t = t.detach().clone()
            
    # Step 5: Finalization
    # Assign best_q and best_t back
    # In a real implementation, these would be assigned to the main pose tensor
    # Here we return the best poses
    optimized_poses = torch.cat([best_q, best_t], dim=1)
    
    return optimized_poses
```

**Edge Cases and Pitfalls:**
*   **Local Minima:** Poor initial pose estimates can lead to convergence in local minima. This is mitigated by the coarse-to-fine strategies in variants like [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) or [CBARF](9066-cbarf.md) [S1, unofficial].
*   **Zero-Valued Pixels:** Background pixels or empty space can result in zero-valued renders, causing division by zero or misleading gradients. The `black_hole_threshold` mask handles this [S1, unofficial].
*   **Quaternion Normalization:** Quaternions must remain normalized. The Adam optimizer does not enforce this constraint automatically. [G] In practice, the quaternion components should be normalized after each optimization step or the optimization should be performed on the tangent space (e.g., using axis-angle or rotation vectors) to maintain unit norm. The provided pseudocode does not explicitly normalize, which is a potential pitfall if the source implementation relies on the optimizer's small steps to maintain approximate normalization.

## Subtopics

### [Best Viewpoint Selection via Opacity Entropy](1121-best-viewpoint-selection-via-opacity-entropy.md)
Optimizes camera parameters to maximize visual information by maximizing the entropy of the rendered alpha channel. It uses raw opacity accumulation rather than normalized visibility, making it sensitive to light absorption/transmission.

### [Radiometric Self-Calibration](1820-radiometric-self-calibration.md)
Optimizes per-camera radiometric parameters (gain and bias) to correct photometric errors. It adjusts the observation model rather than volume properties, improving reconstruction quality by ~1 dB.

### [BARF (Bundle-Adjusting Neural Radiance Fields)](6872-barf-bundle-adjusting-neural-radiance-fields.md)
Reconstructs NeRF from images with erroneous poses using a coarse-to-fine refinement strategy that gradually increases high-frequency positional encoding components to stabilize joint optimization.

### [Joint Optimization of NeRF and Camera Parameters](7492-joint-optimization-of-nerf-and-camera-parameters.md)
Jointly optimizes NeRF and camera parameters (poses and focal length) for forward-facing images, eliminating the need for pre-computed poses via SfM. It initializes poses randomly and converges them simultaneously with the scene.

### [CT-NeRF](8541-ct-nerf.md)
An incremental reconstruction pipeline for sequential RGB images without prior pose input. It handles complex trajectories by constructing a pose graph and using reprojection loss from dense pixel-level correspondences.

### [Levenberg-Marquardt Sparse Variant](8578-levenberg-marquardt-sparse-variant.md)
A sparse variant of the Levenberg-Marquardt algorithm to minimize reprojection error, producing optimal estimates for 3D structure and viewing parameters.

### [Neighbor-Replacement Strategy](8651-neighbor-replacement-strategy.md)
Improves BA results by replacing poorly estimated poses with those from neighboring cameras, identified via a criterion based on rendered image quality.

### [L2G-NeRF](8671-l2g-nerf.md)
Combines local photometric alignment with global geometric alignment. It uses a two-stage process: non-parametric pixel-wise flexible alignment followed by parametric frame-wise global alignment to constrain rigid geometric consistency.

### [CBARF](9066-cbarf.md)
Uses a cascaded bundle-adjustment structure (coarse, recursive, fine stages) to progressively refine poses and rectify errors, allowing convergence from large initial errors where standard methods might diverge.

## Operating characteristics
*   **Time Complexity:** Linear in the number of optimization iterations and the number of pixels/views. Each iteration requires a forward render and a backward pass through the differentiable renderer.
*   **Memory Complexity:** Depends on the rendering method. For differentiable rasterization (e.g., 3DGS), it is relatively low. For volumetric rendering with AD, it can be high ($O(kn)$) unless adjoint or analytic methods are used [44-radiance-field-optimization.md].
*   **Performance:** Effective for refining poses when initial estimates are within a reasonable basin of attraction. For large errors, variants like [CBARF](9066-cbarf.md) or [L2G-NeRF](8671-l2g-nerf.md) are required [S1, unofficial].
*   **Strengths:** Directly minimizes pixel-wise reprojection error; integrates seamlessly with differentiable rendering pipelines; does not require explicit feature matching.
*   **Limitations:** Sensitive to initialization; can converge to local minima; computationally expensive for large datasets if not optimized (e.g., sparse variants).
*   **When to Use:** When camera poses are inaccurate but not completely wrong, or when refining poses after an initial reconstruction.
*   **When Not to Use:** When poses are completely unknown (use [Joint Optimization](7492-joint-optimization-of-nerf-and-camera-parameters.md) or [CT-NeRF](8541-ct-nerf.md) instead); when computational resources are extremely limited.

## Background
Bundle Adjustment is a classical technique in computer vision for refining 3D structure and camera poses. In the context of differentiable volume rendering, it has been adapted to optimize camera parameters alongside neural scene representations. The specific implementation details described here (Adam optimizer with separate learning rates for quaternion and translation, cosine annealing scheduler) are derived from the [InstantSplat](https://github.com/NVlabs/InstantSplat) repository [S1, unofficial]. This approach builds on the broader family of [Radiance-field optimization](44-radiance-field-optimization.md) methods [44-radiance-field-optimization.md].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] Best Viewpoint Selection via Opacity Entropy, https://github.com/... (unofficial)
