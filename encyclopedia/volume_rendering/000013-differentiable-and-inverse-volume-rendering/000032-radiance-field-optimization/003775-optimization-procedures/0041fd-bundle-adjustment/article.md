# Bundle Adjustment

Also called: Bundle Adjustment and Pose Optimization, Camera and Pose Optimization, Pose Optimization via Differentiable Rendering

## Overview
Bundle Adjustment (BA) in the context of differentiable volume rendering and neural radiance fields is an optimization procedure that iteratively refines camera pose parameters (quaternion and translation) for a given view by minimizing the pixel-wise loss between the rendered image and the ground truth image [S1, unofficial]. It solves the problem of inaccurate or missing camera poses, which are typically required inputs for volume rendering but may be noisy or entirely absent in uncalibrated scenarios [S1, unofficial]. Unlike sibling methods that may rely on radiometric calibration or joint optimization of neural fields with complex geometric constraints [Siblings], this method specifically targets the geometric alignment of the camera extrinsics using gradient descent on the differentiable rendering output [S1, unofficial]. It sets itself apart by treating the camera pose as a learnable tensor optimized directly against the photometric error of the volume render, often employing specific learning rate schedules and masking strategies to handle sparse or zero-valued pixels [S1, unofficial].

## How it works
The method optimizes the 6-DoF camera pose (rotation and translation) for individual views or batches of views. It assumes a differentiable rendering pipeline (such as 3D Gaussian Splatting or NeRF) is available, which can accept a modified camera pose and produce a rendered image.

**Inputs and Outputs:**
- **Inputs:**
  - `view`: The camera view parameters (intrinsic and initial extrinsic).
  - `gt`: Ground truth image tensor for the view.
  - `gaussians` / `scene`: The scene representation (e.g., 3D Gaussians or NeRF weights) assumed fixed during pose optimization.
  - `num_iter`: Number of optimization iterations.
  - `black_hole_threshold`: Threshold to mask out invalid pixels (default 0.0).
- **Outputs:**
  - `optimized_pose`: The refined camera pose (quaternion and translation).
  - `rendered_image`: The final image rendered with the optimized pose.

**Data Structures:**
- **Camera Pose Tensor:** A tensor of shape `[7]` or separate tensors for quaternion `[4]` and translation `[3]`.
  - `camera_tensor_q`: Quaternion representing rotation, shape `[4]`.
  - `camera_tensor_T`: Translation vector, shape `[3]`.
  - Combined `camera_pose`: Concatenation of `q` and `T`, shape `[7]`.
- **Optimizer State:** Adam optimizer state for `q` and `T`.

**Numbered Steps:**

1.  **Initialization:**
    - Extract the initial world-view transform from the `view` object.
    - Split the transform into a quaternion `q` (first 4 elements) and translation `T` (last 3 elements).
    - Create PyTorch tensors `camera_tensor_q` and `camera_tensor_T` from these values.
    - Set `requires_grad=True` for both tensors to enable gradient computation.

2.  **Optimizer Setup:**
    - Initialize an Adam optimizer with two parameter groups:
      - Group 1: `camera_tensor_T` (translation) with learning rate `lr_T = 0.003`.
      - Group 2: `camera_tensor_q` (quaternion) with learning rate `lr_q = 0.001`.
    - Set optimizer hyperparameters: `betas=(0.9, 0.999)`, `weight_decay=1e-4`.
    - Initialize a Cosine Annealing Learning Rate scheduler with `T_max = num_iter` and `eta_min = 0.0001`.

3.  **Optimization Loop:**
    - Initialize `current_min_loss` to infinity.
    - Initialize `candidate_q` and `candidate_T` to store the best pose found.
    - For `iteration` in `range(num_iter)`:
        a. **Concatenate Pose:** Create `camera_pose` by concatenating `camera_tensor_q` and `camera_tensor_T`.
        b. **Render:** Call the differentiable renderer `render(view, gaussians, pipeline, background, camera_pose=camera_pose)` to obtain the predicted image `rendering`.
        c. **Masking:** Compute a binary mask `mask` where pixels in `rendering` are greater than `black_hole_threshold` (typically 0.0). Convert to float: `mask = (rendering > black_hole_threshold).float()`.
        d. **Loss Computation:** Compute the masked L1 loss between `rendering` and `gt`:
           $$
           \mathcal{L} = \frac{\sum_{i} \text{mask}_i | \hat{C}_i - C_i |}{\sum_{i} \text{mask}_i}
           $$
           where $\hat{C}_i$ is the rendered pixel color and $C_i$ is the ground truth pixel color.
        e. **Backpropagation:** Call `loss.backward()` to compute gradients with respect to `camera_tensor_q` and `camera_tensor_T`.
        f. **Update:** Call `pose_optimizer.step()` to update the pose tensors.
        g. **Zero Gradients:** Call `pose_optimizer.zero_grad(set_to_none=True)`.
        h. **Tracking Best Pose:**
           - If `iteration == 0`, store `initial_loss = loss.item()`.
           - If `loss.item() < current_min_loss`, update `current_min_loss = loss.item()` and save `candidate_q = camera_tensor_q.detach().clone()` and `candidate_T = camera_tensor_T.detach().clone()`.
        i. **Scheduler Step:** Call `scheduler.step()`.

4.  **Finalization:**
    - After the loop, assign `candidate_q` and `candidate_T` back to `camera_tensor_q` and `camera_tensor_T`.
    - Concatenate the optimal pose and render the final image using the optimized parameters.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def bundle_adjustment_pose_optimization(
    view, 
    gaussians, 
    pipeline, 
    background, 
    gt_image, 
    num_iter=100, 
    black_hole_threshold=0.0
):
    """
    Optimizes camera pose for a single view using differentiable rendering.
    
    Args:
        view: Camera view object containing initial world_view_transform.
        gaussians: Scene representation (e.g., 3D Gaussians).
        pipeline: Rendering pipeline configuration.
        background: Background color tensor.
        gt_image: Ground truth image tensor.
        num_iter: Number of optimization iterations.
        black_hole_threshold: Threshold to mask out zero-valued pixels.
        
    Returns:
        optimized_pose: Tensor of shape [7] (q + T).
        final_render: Rendered image with optimized pose.
    """
    # 1. Initialization
    # Extract initial pose from view's world_view_transform
    # Assuming view.world_view_transform is a 4x4 matrix or similar structure
    # Split into quaternion (4) and translation (3)
    # Note: Exact extraction depends on the specific view class implementation.
    # Here we assume a standard conversion or direct tensor access.
    initial_pose_data = view.get_world_view_transform().detach() # Placeholder for actual extraction logic
    # For the purpose of this generic implementation, we assume we can get q and T
    # In InstantSplat, this is done via specific tensor manipulation of the view's pose.
    
    # Let's assume we have a function to get q and T from the view's pose matrix
    # This is a simplification; actual code might involve matrix decomposition.
    q_init, T_init = extract_q_T_from_view(view) 
    
    camera_tensor_q = torch.tensor(q_init, dtype=torch.float32, requires_grad=True)
    camera_tensor_T = torch.tensor(T_init, dtype=torch.float32, requires_grad=True)
    
    # 2. Optimizer Setup
    pose_optimizer = optim.Adam(
        [
            {'params': [camera_tensor_T], 'lr': 0.003},
            {'params': [camera_tensor_q], 'lr': 0.001}
        ],
        betas=(0.9, 0.999),
        weight_decay=1e-4
    )
    
    scheduler = optim.lr_scheduler.CosineAnnealingLR(
        pose_optimizer, 
        T_max=num_iter, 
        eta_min=0.0001
    )
    
    current_min_loss = float('inf')
    candidate_q = None
    candidate_T = None
    
    # 3. Optimization Loop
    for iteration in range(num_iter):
        # a. Concatenate Pose
        camera_pose = torch.cat([camera_tensor_q, camera_tensor_T])
        
        # b. Render
        # The render function must accept the modified camera_pose
        rendering = render(
            view=view, 
            gaussians=gaussians, 
            pipeline=pipeline, 
            background=background, 
            camera_pose=camera_pose
        )
        
        # c. Masking
        mask = (rendering > black_hole_threshold).float()
        
        # d. Loss Computation (Masked L1)
        # Avoid division by zero if mask is all zeros
        mask_sum = mask.sum()
        if mask_sum > 0:
            loss = (mask * torch.abs(rendering - gt_image)).sum() / mask_sum
        else:
            loss = torch.tensor(0.0, device=rendering.device)
            
        # e. Backpropagation
        loss.backward()
        
        # f. Update
        pose_optimizer.step()
        
        # g. Zero Gradients
        pose_optimizer.zero_grad(set_to_none=True)
        
        # h. Tracking Best Pose
        loss_val = loss.item()
        if iteration == 0:
            initial_loss = loss_val
            
        if loss_val < current_min_loss:
            current_min_loss = loss_val
            candidate_q = camera_tensor_q.detach().clone()
            candidate_T = camera_tensor_T.detach().clone()
            
        # i. Scheduler Step
        scheduler.step()
        
    # 4. Finalization
    camera_tensor_q.copy_(candidate_q)
    camera_tensor_T.copy_(candidate_T)
    
    # Render final image with optimized pose
    final_camera_pose = torch.cat([camera_tensor_q, camera_tensor_T])
    final_render = render(
        view=view, 
        gaussians=gaussians, 
        pipeline=pipeline, 
        background=background, 
        camera_pose=final_camera_pose
    )
    
    return torch.cat([camera_tensor_q, camera_tensor_T]), final_render

def extract_q_T_from_view(view):
    """
    Helper function to extract quaternion and translation from a view object.
    Implementation details depend on the specific view class (e.g., GaussianCamera).
    This is a placeholder for the actual extraction logic found in the source code.
    """
    # In InstantSplat, the view's pose is often stored as a 4x4 matrix or similar.
    # We assume a standard conversion exists or is implemented elsewhere.
    # For the sake of this article, we assume this helper exists and works.
    # [G] This function is a placeholder for the specific matrix decomposition 
    # required by the view class in the source code.
    pass
```

**Parameters:**
- `lr_T` (Translation Learning Rate): 0.003.
- `lr_q` (Quaternion Learning Rate): 0.001.
- `betas`: (0.9, 0.999).
- `weight_decay`: 1e-4.
- `eta_min` (Scheduler Minimum LR): 0.0001.
- `black_hole_threshold`: 0.0.
- `num_iter`: Determined by command-line arguments (e.g., `args.optim_test_pose_iter`).

**Edge Cases and Pitfalls:**
- **Zero-Valued Pixels:** The mask is crucial to prevent gradients from flowing through background or invalid pixels, which can destabilize optimization. If the entire image is masked out, the loss is set to 0.0 [S1, unofficial].
- **Local Minima:** The choice of learning rates and the cosine annealing schedule helps in navigating the loss landscape. The quaternion parameterization requires care to maintain unit norm, though the source code does not explicitly enforce normalization after each step, relying on the small step sizes and Adam optimizer [S1, unofficial]. [G] In practice, quaternion normalization might be applied after optimization or via a specific layer, but the provided pseudocode does not include it.
- **Convergence:** The method tracks the best pose seen during optimization, not just the final one, to avoid overfitting to noisy gradients in later iterations [S1, unofficial].

## Subtopics

### [Best Viewpoint Selection via Opacity Entropy](1121-best-viewpoint-selection-via-opacity-entropy.md)
Best Viewpoint Selection via Opacity Entropy is a method for selecting representative camera viewpoints in volume visualization by maximizing the visual information content of the rendered image. It solves the problem of identifying camera positions that reveal the most structural detail in a volume dataset, which is critical for tasks such as transfer function reconstruction and multi-view initialization. Unlike sibling methods that may rely on radiometric calibration or joint optimization of neural fields [Siblings], this method specifically targets the geometric and density information encoded in the alpha channel of differentiable volume renders. It sets itself apart by using a differentiable entropy metric on the opacity values to guide gradient-based optimization of camera parameters.

### [Radiometric Self-Calibration](1820-radiometric-self-calibration.md)
Radiometric Self-Calibration optimizes radiometric parameters, specifically exposure and sensor bias, during the reconstruction process to correct for photometric errors and intensity mismatches between training views. It addresses the limitation that cameras in a multiview system are often geometrically calibrated but not color-calibrated, leading to inconsistent pixel values for the same radiance across different sensors. Unlike methods that assume fixed camera response, this approach learns per-camera gain and bias parameters to ensure that reconstructed radiance values map consistently to pixel values. Enabling exposure and sensor bias estimation improves reconstruction quality by approximately 1 dB.

### [BARF (Bundle-Adjusting Neural Radiance Fields)](6872-barf-bundle-adjusting-neural-radiance-fields.md)
BARF reconstructs a Neural Radiance Field (NeRF) from multi-view images when the input camera poses contain errors. It differs from standard NeRF by jointly optimizing the radiance field and camera parameters, using a coarse-to-fine strategy to prevent the model from converging to local minima caused by pose inaccuracies.

### [Joint Optimization of NeRF and Camera Parameters](7492-joint-optimization-of-nerf-and-camera-parameters.md)
This method jointly optimizes a Neural Radiance Field (NeRF) model and camera parameters (poses and focal length) for forward-facing images, eliminating the need for pre-computed camera poses via Structure-from-Motion (SfM) systems like COLMAP. It simplifies classical two-stage Novel View Synthesis (NVS) pipelines by treating camera extrinsics and intrinsics as learnable parameters alongside the neural network weights. Unlike [Bundle Adjustment](8540-bundle-adjustment.md), which typically refines known poses, this approach initializes poses randomly or with coarse estimates and converges them simultaneously with the scene representation.

### [CT-NeRF](8541-ct-nerf.md)
CT-NeRF is an incremental reconstruction optimization pipeline that jointly optimizes camera poses and a neural radiance field (NeRF) from sequential RGB images without prior pose or depth input. It is specifically designed to handle complex trajectories involving large rotations where existing methods fail. Unlike its siblings, CT-NeRF escapes local minima by constructing a pose graph with edges between neighboring camera poses (enforcing inter-pose consistency) and by using reprojection loss derived from dense pixel-level correspondences, rather than relying solely on photometric loss.

### [Levenberg-Marquardt Sparse Variant](8578-levenberg-marquardt-sparse-variant.md)
The Levenberg-Marquardt Sparse Variant is a non-linear least-squares optimization technique used to minimize reprojection error between observed and predicted image points, jointly optimizing 3D structure and camera parameters (pose and calibration). It sets itself apart from dense bundle adjustment by exploiting the sparsity of the Jacobian matrix to reduce computational complexity from inverting a $3n \times 3n$ matrix to a $7m \times 7m$ matrix, where $n$ is the number of 3D points and $m$ is the number of cameras. This makes it suitable for scenarios where the number of features significantly exceeds the number of images ($n \gg m$).

### [Neighbor-Replacement Strategy](8651-neighbor-replacement-strategy.md)
The Neighbor-Replacement Strategy is a post-processing step applied within cascaded Bundle Adjustment (BA) modules to correct poorly estimated camera poses. It identifies inferior views based on rendered image quality and replaces their poses with those of the most similar neighboring cameras. Unlike standard BA which refines poses iteratively, this strategy explicitly swaps poses to eliminate the detrimental impact of inaccurate initial estimates on scene representation. It is distinct from sibling methods like [Best Viewpoint Selection via Opacity Entropy](1121-best-viewpoint-selection-via-opacity-entropy.md) by focusing on pose correction via replacement rather than viewpoint selection, and differs from [BARF (Bundle-Adjusting Neural Radiance Fields)](6872-barf-bundle-adjusting-neural-radiance-fields.md) by using a discrete replacement mechanism rather than continuous joint optimization of poses and fields.

### [L2G-NeRF](8671-l2g-nerf.md)
L2G-NeRF is a bundle-adjusting method for Neural Radiance Fields that jointly optimizes neural 3D representations and camera poses by combining a local photometric alignment objective with a global geometric alignment objective. It solves the problem of suboptimal solutions caused by poor initialization, which plagues purely parametric methods, and the inability to handle large displacements, which affects purely non-parametric methods. It sets itself apart by using a two-stage process: first, a pixel-wise flexible alignment (non-parametric) to initialize the solution, followed by a frame-wise constrained parametric alignment (global) to enforce geometric consistency.

### [CBARF](9066-cbarf.md)
CBARF is a 3D reconstruction method that optimizes imperfect camera poses using a cascaded bundle-adjustment (BA) structure to progressively refine poses and rectify erroneous ones. It builds on [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) by introducing a coarse-to-fine optimization strategy with distinct modulation factors for each stage, allowing it to handle larger initial pose errors (e.g., 30.4° rotation deviation) where standard BARF may overfit.

### [Levenberg-Marquardt Pose Optimization](11301-levenberg-marquardt-pose-optimization.md)
Levenberg-Marquardt (LM) Pose Optimization is an iterative numerical method that estimates the 6-DoF camera pose by minimizing the squared error between features extracted from a reference image and features rendered from a current pose guess. It distinguishes itself from gradient-based differentiable rendering approaches by relying on a numerically estimated Jacobian via finite-difference pose perturbations rather than analytical gradients, making it robust to complex lens and illumination dependencies.

## Operating characteristics
- **Time Complexity:** The method runs for a fixed number of iterations `num_iter`. Each iteration involves a forward pass through the differentiable renderer and a backward pass. The complexity is $O(\text{num\_iter} \cdot (N_{\text{render}} + N_{\text{grad}}))$, where $N_{\text{render}}$ is the cost of rendering and $N_{\text{grad}}$ is the cost of backpropagation.
- **Memory Complexity:** The memory usage is dominated by the scene representation (e.g., 3D Gaussians or NeRF weights) and the intermediate activations during the forward and backward passes. The pose optimization itself adds minimal overhead ($O(1)$ for the pose tensors).
- **Measured Performance:** The specific performance metrics (PSNR, SSIM) depend on the scene and the initial pose accuracy. The method is designed to refine poses to improve rendering quality, particularly in test sets where poses might be less accurate or unavailable [S1, unofficial].
- **Strengths:** Simple to implement, leverages existing differentiable rendering pipelines, effective for refining known poses or correcting small errors.
- **Limitations:** Sensitive to initial pose accuracy; may converge to local minima if the initial pose is far from the true pose. Does not jointly optimize the scene representation, which may lead to inconsistencies if the scene is not well-initialized.
- **When to Use:** When camera poses are known but inaccurate, or when a quick refinement of poses is needed for a fixed scene representation.
- **When Not to Use:** When poses are completely unknown or when the scene representation is also poorly initialized, in which case joint optimization methods like BARF or CBARF may be more appropriate.

## Background
Bundle Adjustment is a classical technique in computer vision for optimizing camera poses and 3D structure simultaneously [S1, unofficial]. In the context of differentiable volume rendering and neural radiance fields, it has been adapted to optimize camera poses using gradient descent on the rendered image loss. The specific implementation described here is derived from the InstantSplat project, which applies BA to 3D Gaussian Splatting [S1, unofficial]. The method builds on the principles of differentiable rendering, where gradients can be propagated through the rendering process to update scene and camera parameters [44-radiance-field-optimization.md].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] Best Viewpoint Selection via Opacity Entropy, https://doi.org/10.1109/TVCG.2020.3003688, fetched 2026-10-27
[S3] Best Viewpoint Selection via Opacity Entropy, https://arxiv.org/abs/2006.04898, fetched 2026-10-27 (unofficial)
