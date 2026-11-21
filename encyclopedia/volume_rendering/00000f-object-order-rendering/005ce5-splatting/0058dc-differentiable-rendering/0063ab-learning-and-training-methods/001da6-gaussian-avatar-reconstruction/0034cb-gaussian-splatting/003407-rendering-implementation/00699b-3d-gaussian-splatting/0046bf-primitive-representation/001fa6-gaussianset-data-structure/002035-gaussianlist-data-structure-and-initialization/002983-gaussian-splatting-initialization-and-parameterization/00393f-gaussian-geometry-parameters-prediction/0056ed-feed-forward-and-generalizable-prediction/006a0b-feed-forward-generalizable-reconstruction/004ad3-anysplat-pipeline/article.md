# AnySplat Pipeline

Also called: Feed-forward 3D Gaussian Splatting from Unconstrained Views

## Overview
The AnySplat Pipeline is a feed-forward method for generalizable 3D Gaussian Splatting that operates on uncalibrated input views, simultaneously predicting Gaussian parameters, depth maps, and camera poses without per-scene optimization [S1, unofficial]. It distinguishes itself from sibling methods like [Splatt3R](16423-splatt3r.md) and [ZeroGS](16425-zerogs.md) by explicitly integrating a Differentiable Voxelization module to convert pixel-wise Gaussians into a voxelized representation, which stabilizes rendering and enables robust geometry supervision via pseudo-geometry priors from a pretrained VGGT model [S1, unofficial].

## How it works
The pipeline maps a set of input images directly to a 3D Gaussian scene representation through a transformer-based encoder and decoder architecture.

**Inputs and Outputs**
*   **Input:** A batch of $K$ uncalibrated RGB images $I \in \mathbb{R}^{K \times 3 \times H \times W}$, typically with $H=W=448$. The input tensor shape is $[1, K, 3, 448, 448]$ (batch, views, channels, height, width) [S1, unofficial].
*   **Outputs:**
    *   A set of 3D Gaussians $\mathcal{G}$ defined by mean $\mu$, scale $s$, rotation $r$, and color $c$.
    *   Predicted camera poses (extrinsic $E$ and intrinsic $K$ matrices) for the input views [S1, unofficial].
    *   Predicted depth maps $D$ for the input views [S1, unofficial].

**Data Structures**
The Gaussian parameters are predicted per-pixel from the feature map. Each pixel corresponds to a 3D Gaussian primitive. The parameters include:
*   $\mu \in \mathbb{R}^3$: 3D position.
*   $s \in \mathbb{R}^3$: Scale (log-space or linear).
*   $r \in \mathbb{R}^4$: Rotation (quaternion).
*   $c$: Color coefficients (e.g., SH coefficients or RGB).

**Algorithm Steps**
1.  **Feature Encoding:** The input images $I$ are normalized to $[-1, 1]$ range (via $(I+1)*0.5$ if inputs are in $[0,1]$) and passed through a transformer-based geometry encoder to extract multi-scale features [S1, unofficial].
2.  **Parameter Prediction:** Three decoder heads process the encoded features:
    *   $F_G$: Predicts Gaussian parameters $(\mu, \sigma, r, s, c)$.
    *   $F_D$: Predicts a dense depth map $D$.
    *   $F_C$: Predicts camera poses $p$ (extrinsics and intrinsics) [S1, unofficial].
3.  **Gaussian Construction:** The predicted parameters are used to construct a set of pixel-wise 3D Gaussians. Each pixel in the input view generates a Gaussian centered at the back-projected 3D point, with orientation and scale derived from the network output [S1, unofficial].
4.  **Differentiable Voxelization:** The pixel-wise Gaussians are voxelized into a pre-voxel 3D Gaussian representation using a Differentiable Voxelization module. This step aggregates Gaussians into a voxel grid, resolving density issues and providing a structured representation for rendering [S1, unofficial].
5.  **Rendering:** Multi-view images and depth maps are rendered from the voxelized Gaussians using differentiable rasterization [S1, unofficial].
6.  **Loss Computation:**
    *   **RGB Loss:** The rendered images are compared against ground truth images using an RGB loss (e.g., L1 or LPIPS) [S1, unofficial].
    *   **Geometry Loss:** The rendered depth maps, the decoded depth $D$, and the predicted poses $p$ are used to compute geometry losses. These losses are supervised by pseudo-geometry priors obtained from a pretrained VGGT model, which provides consistent depth and pose constraints for uncalibrated inputs [S1, unofficial].
7.  **Inference:** During inference, the model takes normalized images and outputs the Gaussians and predicted poses. The code snippet `gaussians, pred_context_pose = model.inference((images+1)*0.5)` demonstrates the standard inference call [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def anysplat_pipeline(images: torch.Tensor, model: torch.nn.Module) -> tuple:
    """
    Runs the AnySplat pipeline on input images.
    
    Args:
        images: Input tensor of shape [B, K, 3, 448, 448] with values in [0, 1].
        model: The trained AnySplat model containing encoder and decoder heads.
        
    Returns:
        gaussians: Predicted 3D Gaussian parameters.
        pred_context_pose: Dictionary containing predicted extrinsic and intrinsic matrices.
    """
    # Normalize images to [-1, 1] as per the inference script
    normalized_images = (images + 1.0) * 0.5
    
    # Run the model inference
    # The model internally handles encoding, decoding, and voxelization
    gaussians, pred_context_pose = model.inference(normalized_images)
    
    return gaussians, pred_context_pose

def compute_losses(rendered_images, rendered_depths, gt_images, gt_depths, 
                   pred_depth, pred_poses, model_vggts):
    """
    Computes the total loss for training.
    """
    # RGB Loss
    rgb_loss = torch.nn.functional.l1_loss(rendered_images, gt_images)
    
    # Geometry Losses
    # Supervised by pseudo-geometry priors from pretrained VGGT
    # This involves comparing rendered depth, decoded depth D, and poses p
    # against the priors provided by VGGT
    geometry_loss = compute_geometry_loss(rendered_depths, pred_depth, pred_poses, model_vggts)
    
    total_loss = rgb_loss + geometry_loss
    return total_loss
```

**Edge Cases and Pitfalls**
*   **Uncalibrated Inputs:** Since the method predicts poses, incorrect pose predictions can lead to geometric inconsistencies. The use of VGGT priors helps mitigate this but may fail on highly ambiguous scenes [S1, unofficial].
*   **Voxelization Artifacts:** The Differentiable Voxelization step may introduce artifacts if the voxel resolution is not chosen appropriately. Too coarse a voxel grid loses detail; too fine a grid increases memory usage [S1, unofficial].
*   **Precision:** The model relies on PyTorch 2.2.0 and CUDA 12.1 for optimal performance. Using older versions may lead to compatibility issues with the differentiable voxelization module [S1, unofficial].

## Subtopics

### [SubSplat](13524-subsplat.md)
A high-resolution variant that uses a Sub-pixel Gaussian Reparameterizer to increase Gaussian density from low-resolution features.

### [Deformable Radial Kernel (DRK)](13699-deformable-radial-kernel-drk.md)
A variant that replaces radial Gaussians with planar kernels for better geometric fidelity and fewer primitives.

## Operating characteristics
*   **Time Complexity:** The method is feed-forward, so inference time is dominated by the transformer encoder and decoder. It is significantly faster than per-scene optimization methods like original 3D Gaussian Splatting.
*   **Memory Complexity:** Memory usage depends on the number of input views $K$ and the voxel grid resolution. The voxelization step adds overhead compared to pure pixel-wise methods.
*   **Performance:** The method achieves competitive novel-view synthesis quality on uncalibrated inputs, leveraging VGGT priors for robust geometry [S1, unofficial].
*   **Strengths:** Handles uncalibrated inputs, provides explicit pose and depth predictions, and uses voxelization for stable rendering.
*   **Limitations:** Relies on the quality of the pretrained VGGT model for geometry supervision. Performance may degrade on scenes with significant textureless regions or extreme viewpoints.

## Background
The AnySplat Pipeline was introduced by InternRobotics as a feed-forward approach to 3D Gaussian Splatting that generalizes to unconstrained views [S1, unofficial]. It builds on the principles of generalizable reconstruction by predicting scene parameters directly from images, while addressing the challenges of uncalibrated inputs through pose prediction and pseudo-geometry supervision.

## Sources
[S1] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
