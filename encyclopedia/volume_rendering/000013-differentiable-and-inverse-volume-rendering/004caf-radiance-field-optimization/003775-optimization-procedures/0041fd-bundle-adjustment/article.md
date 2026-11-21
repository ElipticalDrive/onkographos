# Bundle Adjustment

Also called: Bundle Adjustment and Pose Optimization, Camera and Pose Optimization, Pose Optimization via Differentiable Rendering

## Overview
Bundle Adjustment is an iterative optimization strategy that refines camera poses (quaternion and translation) for a given view by minimizing the pixel-wise loss between the rendered image and the ground truth image using gradient descent [S1, unofficial]. It solves the problem of inaccurate initial camera poses by jointly optimizing the pose parameters alongside the scene representation (such as Gaussian Splatting or NeRF) to ensure geometric consistency [S1, unofficial]. Unlike sibling methods that may rely on pre-computed Structure-from-Motion (SfM) poses or purely parametric alignment, this method explicitly treats pose as a learnable parameter optimized via differentiable rendering gradients [S1, unofficial]. It sets itself apart by applying per-view optimization loops that refine poses iteratively, often using specific learning rate schedules and masking strategies to handle sparse or noisy renderings [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:**
    - `gaussians`: The scene representation (e.g., 3D Gaussians) with fixed parameters.
    - `view`: The camera view parameters (initial pose estimate).
    - `gt`: The ground truth image for the view.
    - `num_iter`: The number of optimization iterations.
    - `black_hole_threshold`: Threshold to mask out zero-valued pixels (default 0.0) [S1, unofficial].
- **Outputs:**
    - `optimized_pose`: The refined camera pose (quaternion and translation) that minimizes the rendering loss.

**Data Structures:**
- **Camera Pose Tensor:** A tensor of size 7, initialized from the view's `world_view_transform`.
    - Indices 0-3: Quaternion ($q$) representing rotation.
    - Indices 4-6: Translation vector ($T$) representing position.
- **Optimizer State:** Adam optimizer state for $q$ and $T$.

**Parameters:**
- **Learning Rates:**
    - Translation ($T$): $lr_T = 0.003$ [S1, unofficial].
    - Quaternion ($q$): $lr_q = 0.001$ [S1, unofficial].
- **Optimizer Hyperparameters:**
    - Betas: $(\beta_1, \beta_2) = (0.9, 0.999)$ [S1, unofficial].
    - Weight Decay: $\lambda = 10^{-4}$ [S1, unofficial].
- **Scheduler:**
    - Type: Cosine Annealing [S1, unofficial].
    - $T_{max}$: `num_iter` [S1, unofficial].
    - $\eta_{min}$: $0.0001$ [S1, unofficial].

**Steps:**
1. **Initialization:**
    - Extract the initial camera pose from the view's `world_view_transform`.
    - Split the pose tensor into quaternion $q$ (first 4 elements) and translation $T$ (last 3 elements).
    - Set `requires_grad=True` for both $q$ and $T$ to enable gradient computation [S1, unofficial].
    - Initialize the Adam optimizer with two parameter groups:
        - Group 1: $T$ with $lr = 0.003$.
        - Group 2: $q$ with $lr = 0.001$.
    - Initialize the Cosine Annealing LR scheduler with $T_{max} = \text{num\_iter}$ and $\eta_{min} = 0.0001$ [S1, unofficial].
    - Set `current_min_loss` to infinity and `candidate_q`, `candidate_T` to None [G].
    - Set `initial_loss` to None [G].

2. **Optimization Loop:**
    - For each iteration $i$ from 0 to `num_iter` - 1:
        - Concatenate $q$ and $T$ into a single `camera_pose` tensor of size 7.
        - Call the differentiable rendering function:
          $$ \hat{C} = \text{render}(\text{view}, \text{gaussians}, \text{pipeline}, \text{background}, \text{camera\_pose}) $$
          where $\hat{C}$ is the rendered image [S1, unofficial].
        - Compute a binary mask $M$ to exclude zero-valued (background/empty) pixels:
          $$ M = (\hat{C} > \text{black\_hole\_threshold}).\text{float}() $$
          where `black_hole_threshold` is typically 0.0 [S1, unofficial].
        - Compute the photometric loss using the mask:
          $$ L = \text{L1Loss}(\hat{C}, C_{gt}, M) $$
          where $C_{gt}$ is the ground truth image. The L1 loss is computed only on pixels where $M=1$ [S1, unofficial].
        - Perform backpropagation:
          $$ \nabla_{q,T} L = \frac{\partial L}{\partial q, \partial T} $$
        - Update parameters:
          $$ \text{optimizer.step}() $$
        - Zero gradients:
          $$ \text{optimizer.zero\_grad}(\text{set\_to\_none}=\text{True}) $$
        - Update scheduler:
          $$ \text{scheduler.step}() $$
        - **Tracking Best Pose:**
            - If $i = 0$, store $L$ as `initial_loss` [S1, unofficial].
            - If $L < \text{current\_min\_loss}$:
                - Update $\text{current\_min\_loss} = L$.
                - Save `candidate_q` = $q$ and `candidate_T` = $T$ [S1, unofficial].

3. **Finalization:**
    - After the loop, assign the best found parameters back to the camera tensor:
      $$ q_{final} = \text{candidate\_q}, \quad T_{final} = \text{candidate\_T} $$
    - Concatenate $q_{final}$ and $T_{final}$ to form the optimized pose [S1, unofficial].
    - Render the final image using the optimized pose [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
from torch.optim import Adam
from torch.optim.lr_scheduler import CosineAnnealingLR

def bundle_adjustment_pose_optimization(
    gaussians,
    view,
    gt_image,
    pipeline,
    background_color,
    num_iter,
    black_hole_threshold=0.0
):
    """
    Optimizes camera pose (quaternion and translation) for a single view.
    
    Args:
        gaussians: Scene representation (e.g., GaussianModel).
        view: Camera view object containing initial pose.
        gt_image: Ground truth image tensor.
        pipeline: Rendering pipeline parameters.
        background_color: Background color tensor (e.g., [1,1,1] or [0,0,0]).
        num_iter: Number of optimization iterations.
        black_hole_threshold: Threshold to mask out zero-valued pixels.
        
    Returns:
        optimized_pose: Tensor of size 7 [q0, q1, q2, q3, tx, ty, tz].
    """
    # 1. Initialization
    # Extract initial pose from view's world_view_transform
    # Assuming view provides a method or attribute to get the initial pose tensor
    initial_pose_tensor = view.get_world_view_transform() # Placeholder for specific API call
    
    # Split into quaternion (first 4) and translation (last 3)
    q = initial_pose_tensor[0:4].clone().detach().requires_grad_(True)
    T = initial_pose_tensor[4:7].clone().detach().requires_grad_(True)
    
    # Initialize Optimizer
    optimizer = Adam([
        {'params': [T], 'lr': 0.003},
        {'params': [q], 'lr': 0.001}
    ], betas=(0.9, 0.999), weight_decay=1e-4)
    
    # Initialize Scheduler
    scheduler = CosineAnnealingLR(optimizer, T_max=num_iter, eta_min=0.0001)
    
    # Tracking variables
    current_min_loss = float('inf')
    candidate_q = None
    candidate_T = None
    initial_loss = None
    
    # 2. Optimization Loop
    for i in range(num_iter):
        # Concatenate pose
        camera_pose = torch.cat([q, T], dim=0)
        
        # Render
        # Note: render function signature depends on specific implementation (e.g., InstantSplat)
        # It takes gaussians, pipeline, background, and camera_pose
        rendered_image = render(
            view=view, 
            gaussians=gaussians, 
            pipeline=pipeline, 
            background=background_color, 
            camera_pose=camera_pose
        )
        
        # Compute Mask
        # Mask out pixels below the threshold (e.g., black holes/empty space)
        mask = (rendered_image > black_hole_threshold).float()
        
        # Compute Loss (L1 Loss with Mask)
        # Only compute loss where mask is 1
        loss = nn.functional.l1_loss(rendered_image * mask, gt_image * mask)
        
        # Backpropagation
        loss.backward()
        
        # Step Optimizer
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
        
        # Step Scheduler
        scheduler.step()
        
        # Track Best Pose
        if i == 0:
            initial_loss = loss.item()
            
        if loss.item() < current_min_loss:
            current_min_loss = loss.item()
            # Clone to save state
            candidate_q = q.detach().clone()
            candidate_T = T.detach().clone()
            
    # 3. Finalization
    # Assign best found parameters
    q_final = candidate_q
    T_final = candidate_T
    
    # Concatenate to form final optimized pose
    optimized_pose = torch.cat([q_final, T_final], dim=0)
    
    return optimized_pose
```

## Subtopics

### [Best Viewpoint Selection via Opacity Entropy](1121-best-viewpoint-selection-via-opacity-entropy.md)
Best Viewpoint Selection via Opacity Entropy is a method for selecting representative camera viewpoints in volume visualization by maximizing the visual information content of the rendered image. It solves the problem of identifying camera positions that reveal the most structural detail in a volume dataset, which is critical for tasks such as transfer function reconstruction and multi-view initialization. Unlike sibling methods that may rely on radiometric calibration or joint optimization of neural fields [Siblings], this method specifically targets the geometric and density information encoded in the alpha channel of differentiable volume renders. It sets itself apart by using a differentiable entropy metric on the opacity values to guide gradient-based optimization of camera parameters.

### [Radiometric Self-Calibration](1820-radiometric-self-calibration.md)
Radiometric Self-Calibration optimizes radiometric parameters, specifically exposure and sensor bias, during the reconstruction process to correct for photometric errors and intensity mismatches between training views. It addresses the limitation that cameras in a multiview system are often geometrically calibrated but not color-calibrated, leading to inconsistent pixel values for the same radiance across different sensors. Unlike methods that assume fixed camera response, this approach learns per-camera gain and bias parameters to ensure that reconstructed radiance values map consistently to pixel values. Enabling exposure and sensor bias estimation improves reconstruction quality by approximately 1 dB.

### [BARF (Bundle-Adjusting Neural Radiance Fields)](6872-barf-bundle-adjusting-neural-radiance-fields.md)
BARF reconstructs a Neural Radiance Field (NeRF) and refines inaccurate camera poses given multi-view images with pose errors. It differs from standard NeRF optimization by decoupling the optimization of high-frequency scene details from camera pose updates, preventing the optimizer from getting trapped in local minima caused by the high-frequency positional encoding.

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
Levenberg-Marquardt (LM) Pose Optimization is an iterative procedure that minimizes the squared error between reference image features and rendered image features by using a numerically estimated Jacobian. It sets itself apart from its siblings by relying on finite-difference Jacobian approximation via pose perturbation rather than analytical derivatives or derivative-free search, balancing the stability of gradient descent with the speed of Gauss-Newton methods.

### [Token-level Bundle Adjustment](17341-token-level-bundle-adjustment.md)
Token-level Bundle Adjustment (TokenBA) is a per-scene refinement procedure that fits a frozen ZipSplat decoder to input views by jointly optimizing camera poses and scene tokens, recovering fine detail and correcting initial pose errors in a few seconds. It exploits the persistent 2D–3D correspondences established by the cone loss to refine cameras via robust Perspective-n-Point (PnP) and update scene tokens through the frozen decoder using photometric error. Unlike methods that optimize raw Gaussians or require pose supervision, TokenBA achieves faster convergence and higher quality by leveraging the built-in correspondence of scene tokens.

## Operating characteristics
- **Time Complexity:** The method requires $O(\text{num\_iter} \times N)$ operations per view, where $N$ is the number of pixels and samples involved in the differentiable rendering step. Each iteration involves a forward pass (rendering) and a backward pass (gradient computation) [S1, unofficial].
- **Memory Complexity:** Memory usage is dominated by the storage of intermediate states during the forward and backward passes of the differentiable renderer. For Gaussian Splatting, this is generally lower than NeRF due to the explicit representation, but still scales with image resolution and number of Gaussians [S1, unofficial].
- **Measured Performance:** The method is effective for refining poses with initial errors, but performance depends heavily on the initial pose quality and the learning rate schedule [S1, unofficial].
- **Strengths:**
    - Explicitly optimizes pose parameters using gradient descent, allowing for fine-tuning of camera extrinsics [S1, unofficial].
    - Can be integrated into existing differentiable rendering pipelines (e.g., InstantSplat) with minimal code changes [S1, unofficial].
    - Uses masking to ignore background/empty pixels, improving robustness to sparse scenes [S1, unofficial].
- **Limitations:**
    - Computationally expensive due to the iterative nature and the need for multiple forward/backward passes [S1, unofficial].
    - Sensitive to initial pose estimates; large errors may require more iterations or specialized strategies (e.g., coarse-to-fine) [S1, unofficial].
    - May converge to local minima if the scene has ambiguous geometry or insufficient texture [S1, unofficial].
- **When to Use:**
    - When initial camera poses are inaccurate or unavailable.
    - When high-quality novel view synthesis is required and pose refinement is critical.
    - When using differentiable rendering frameworks that support gradient-based optimization of camera parameters [S1, unofficial].
- **When Not to Use:**
    - When computational resources are limited and real-time performance is required.
    - When camera poses are already accurate and well-calibrated.
    - When the scene is highly dynamic or non-rigid, as static pose optimization may not be appropriate [S1, unofficial].

## Background
Bundle Adjustment in the context of differentiable volume rendering and neural radiance fields has emerged as a critical technique for improving the geometric accuracy of 3D reconstructions. The method is closely related to traditional bundle adjustment in computer vision, which jointly optimizes 3D points and camera parameters to minimize reprojection error [S1, unofficial]. In the context of neural rendering, it has been adapted to optimize camera poses by minimizing photometric loss between rendered and ground truth images [S1, unofficial]. The specific implementation described here, involving Adam optimization with cosine annealing and masking strategies, is popularized by frameworks like InstantSplat, which leverage 3D Gaussian Splatting for fast and high-quality reconstruction [S1, unofficial]. The approach addresses the limitations of pre-computed SfM poses, which can be noisy or inaccurate, by refining them end-to-end with the scene representation [S1, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
