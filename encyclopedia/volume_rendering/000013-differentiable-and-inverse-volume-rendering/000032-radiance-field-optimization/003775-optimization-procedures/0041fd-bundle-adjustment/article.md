# Bundle Adjustment

Also called: Bundle Adjustment and Pose Optimization, Camera and Pose Optimization, Pose Optimization via Differentiable Rendering

## Overview
Bundle Adjustment (BA) in the context of differentiable volume rendering iteratively refines camera pose parameters (quaternion and translation) for a given view by minimizing the pixel-wise loss between the rendered image and the ground truth image using gradient descent [S1, unofficial]. It solves the problem of inaccurate initial camera poses, which can lead to poor scene reconstruction or novel view synthesis artifacts. Unlike sibling methods that may rely on radiometric calibration or joint optimization of neural fields [Siblings], this method specifically targets the geometric alignment by treating camera poses as learnable parameters within the differentiable rendering pipeline. It sets itself apart by applying gradient-based optimization directly to the pose tensor to minimize photometric error, often serving as a refinement step or a standalone initialization correction mechanism.

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - Initial camera pose tensor (typically derived from `world_view_transform`), split into quaternion ($q$) and translation ($T$).
  - Scene representation (e.g., 3D Gaussians or NeRF).
  - Ground truth image ($I_{gt}$) for the target view.
  - Rendering pipeline parameters.
- **Outputs:**
  - Optimized camera pose ($q^*, T^*$) that minimizes the reconstruction loss.
  - Rendered image using the optimized pose.

**Data Structures:**
- **Camera Pose Tensor:** A tensor of shape $[7]$ or separate tensors for $q \in \mathbb{R}^4$ and $T \in \mathbb{R}^3$. The quaternion must be normalized during optimization or enforced via projection.
- **Optimizer State:** Adam optimizer state for $q$ and $T$ with separate learning rates.

**Numbered Steps:**
1. **Initialization:** Extract the initial camera pose from the view's `world_view_transform`. Split this into a quaternion component ($q$) and a translation component ($T$). Set `requires_grad=True` for both components to enable gradient computation [S1, unofficial].
2. **Optimizer Setup:** Create an Adam optimizer with two parameter groups:
   - Translation ($T$) with learning rate $lr_T = 0.003$.
   - Quaternion ($q$) with learning rate $lr_q = 0.001$.
   - Hyperparameters: Betas $(\beta_1, \beta_2) = (0.9, 0.999)$, weight decay $\lambda = 10^{-4}$ [S1, unofficial].
3. **Scheduler Setup:** Create a Cosine Annealing Learning Rate scheduler with $T_{max} = N_{iter}$ (number of iterations) and $\eta_{min} = 0.0001$ [S1, unofficial].
4. **Optimization Loop:** For each iteration $k$ from $0$ to $N_{iter}-1$:
   a. Concatenate $q$ and $T$ into a single `camera_pose` tensor.
   b. Call the differentiable renderer: $\hat{I} = \text{render}(view, \text{scene}, \text{pipeline}, \text{background}, \text{camera\_pose})$.
   c. Compute a binary mask $M$ where $M_{ij} = 1$ if $\hat{I}_{ij} > \epsilon$ (black hole threshold, typically $0.0$), else $0$.
   d. Compute the loss: $L = \text{L1Loss}(\hat{I} \odot M, I_{gt} \odot M)$.
   e. Perform backpropagation: $L.\text{backward()}$.
   f. Update parameters: $\text{optimizer.step()}$.
   g. Zero gradients: $\text{optimizer.zero_grad(set\_to\_none=True)}$.
   h. Update scheduler: $\text{scheduler.step()}$.
   i. Track the best pose: If $k=0$, store $L_{min} = L$. If $L < L_{min}$, update $L_{min} = L$ and save candidate $q^*, T^*$ [S1, unofficial].
5. **Finalization:** After the loop, assign the best candidate $q^*$ and $T^*$ back to the camera tensor. Concatenate the optimal pose and render the final image [S1, unofficial].

**Formulas:**
- **Loss Function:**
  $$
  L = \frac{1}{|M|} \sum_{i,j} M_{ij} | \hat{I}_{ij} - I_{gt,ij} |
  $$
  where $\hat{I}$ is the rendered image, $I_{gt}$ is the ground truth, and $M$ is the mask excluding zero-valued (background/black hole) pixels [S1, unofficial].
- **Pose Representation:**
  The camera pose is represented as a concatenation of a unit quaternion $q \in \mathbb{R}^4$ and a translation vector $T \in \mathbb{R}^3$. The renderer interprets this pose to transform the scene coordinates into camera space [S1, unofficial].

**Parameters:**
- $lr_T$: Learning rate for translation. Typical value: $0.003$ [S1, unofficial].
- $lr_q$: Learning rate for quaternion. Typical value: $0.001$ [S1, unofficial].
- $N_{iter}$: Number of optimization iterations. Value determined by configuration (e.g., `args.optim_test_pose_iter`) [S1, unofficial].
- $\eta_{min}$: Minimum learning rate for cosine annealing. Typical value: $0.0001$ [S1, unofficial].
- $\epsilon$: Black hole threshold. Typical value: $0.0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Quaternion Normalization:** Quaternions must remain unit length. If the optimizer updates $q$ such that $\|q\| \neq 1$, the pose becomes invalid. The source implementation does not explicitly mention normalization steps within the loop, implying the renderer or the small step sizes maintain validity, or normalization is applied post-step [G].
- **Local Minima:** Gradient descent on pose can get stuck in local minima if the initial pose is too far from the ground truth. This method is often used for refinement rather than coarse initialization [S1, unofficial].
- **Black Holes:** Zero-valued pixels in the rendered image (due to empty space or clipping) should be masked out to prevent the optimizer from trying to match background noise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.optim.lr_scheduler import CosineAnnealingLR

def bundle_adjustment_pose_optimization(
    initial_pose_tensor: torch.Tensor,
    render_function: callable,
    gt_image: torch.Tensor,
    num_iter: int,
    black_hole_threshold: float = 0.0
) -> tuple[torch.Tensor, torch.Tensor]:
    """
    Optimizes camera pose (quaternion and translation) to minimize pixel-wise loss.
    
    Args:
        initial_pose_tensor: Tensor containing initial pose [q1, q2, q3, q4, tx, ty, tz].
        render_function: Function that takes view, scene, pipeline, background, camera_pose and returns rendered image.
        gt_image: Ground truth image tensor.
        num_iter: Number of optimization iterations.
        black_hole_threshold: Threshold to mask out zero-valued pixels.
        
    Returns:
        best_q: Best quaternion tensor.
        best_T: Best translation tensor.
    """
    # 1. Initialize
    q = initial_pose_tensor[:4].clone().detach().requires_grad_(True)
    T = initial_pose_tensor[4:].clone().detach().requires_grad_(True)
    
    # 2. Optimizer Setup
    optimizer = optim.Adam([
        {'params': [T], 'lr': 0.003},
        {'params': [q], 'lr': 0.001}
    ], betas=(0.9, 0.999), weight_decay=1e-4)
    
    # 3. Scheduler Setup
    scheduler = CosineAnnealingLR(optimizer, T_max=num_iter, eta_min=0.0001)
    
    # 4. Optimization Loop
    best_loss = float('inf')
    best_q = q.clone().detach()
    best_T = T.clone().detach()
    
    for k in range(num_iter):
        # Concatenate pose
        camera_pose = torch.cat([q, T])
        
        # Render
        # Note: render_function signature assumed to be:
        # render(view, scene, pipeline, background, camera_pose=camera_pose)
        rendered_image = render_function(
            view=None, # Placeholder for view object if needed
            scene=None, # Placeholder for scene object if needed
            pipeline=None, # Placeholder for pipeline if needed
            background=None, # Placeholder for background if needed
            camera_pose=camera_pose
        )
        
        # Compute Mask
        mask = (rendered_image > black_hole_threshold).float()
        
        # Compute Loss (L1)
        loss = torch.mean(mask * torch.abs(rendered_image - gt_image))
        
        # Backpropagation
        loss.backward()
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
        
        # Update Scheduler
        scheduler.step()
        
        # Track Best Pose
        if k == 0:
            best_loss = loss.item()
            best_q = q.clone().detach()
            best_T = T.clone().detach()
        elif loss.item() < best_loss:
            best_loss = loss.item()
            best_q = q.clone().detach()
            best_T = T.clone().detach()
            
    return best_q, best_T
```

## Subtopics

### [Best Viewpoint Selection via Opacity Entropy](1121-best-viewpoint-selection-via-opacity-entropy.md)
Best Viewpoint Selection via Opacity Entropy is a method for selecting representative camera viewpoints in volume visualization by maximizing the visual information content of the rendered image. It solves the problem of identifying camera positions that reveal the most structural detail in a volume dataset, which is critical for tasks such as transfer function reconstruction and multi-view initialization. Unlike sibling methods that may rely on radiometric calibration or joint optimization of neural fields [Siblings], this method specifically targets the geometric and density information encoded in the alpha channel of differentiable volume renders. It sets itself apart by using a differentiable entropy metric on the opacity values to guide gradient-based optimization of camera parameters.

### [Radiometric Self-Calibration](1820-radiometric-self-calibration.md)
Radiometric Self-Calibration optimizes radiometric parameters, specifically exposure and sensor bias, during volume reconstruction to correct for photometric errors arising from uncalibrated camera sensors or inconsistent lighting. It differs from geometric self-calibration by focusing on intensity mapping rather than spatial pose, and from simple color correction by learning these parameters jointly with the volume representation.

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
- **Time Complexity:** Linear in the number of iterations $N_{iter}$ and the number of pixels $P$. Each iteration requires a forward pass through the renderer and a backward pass, both $O(P \cdot S)$ where $S$ is samples per pixel.
- **Memory Complexity:** $O(P \cdot S)$ to store intermediate rendering states for backpropagation, plus $O(1)$ for pose parameters.
- **Performance:** Effective for refining poses when initial estimates are close to the ground truth. Performance degrades if initial poses are significantly off, leading to convergence to local minima.
- **Strengths:** Simple to implement using standard autodiff frameworks. Directly minimizes photometric error.
- **Limitations:** Sensitive to initialization. Requires differentiable rendering. Computationally expensive for high-resolution images or large numbers of views if done jointly for all views.
- **When to Use:** When camera poses are known approximately but need refinement, or when used as a post-processing step for pose correction.
- **When Not to Use:** When initial poses are very inaccurate (use coarse-to-fine methods like BARF or CBARF), or when computational resources are extremely limited.

## Background
Bundle Adjustment is a classical technique in computer vision for optimizing camera poses and 3D structure simultaneously [S1, unofficial]. In the context of differentiable volume rendering and neural radiance fields, it has been adapted to optimize camera poses by minimizing the photometric loss between rendered and ground truth images. The specific implementation described here is derived from the InstantSplat framework, which applies this technique to 3D Gaussian Splatting [S1, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
