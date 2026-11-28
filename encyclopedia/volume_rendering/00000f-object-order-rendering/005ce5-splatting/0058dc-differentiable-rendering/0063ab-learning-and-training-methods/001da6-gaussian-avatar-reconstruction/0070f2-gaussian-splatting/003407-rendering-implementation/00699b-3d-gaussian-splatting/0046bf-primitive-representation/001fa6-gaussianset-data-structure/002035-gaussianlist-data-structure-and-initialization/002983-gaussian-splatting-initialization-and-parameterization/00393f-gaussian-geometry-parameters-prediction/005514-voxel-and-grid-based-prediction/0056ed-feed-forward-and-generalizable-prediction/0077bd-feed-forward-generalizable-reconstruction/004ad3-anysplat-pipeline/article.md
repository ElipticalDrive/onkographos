# AnySplat Pipeline

Also called: Feed-forward 3D Gaussian Splatting from Unconstrained Views

## Overview
AnySplat is a feed-forward, generalizable reconstruction method that predicts a set of 3D Gaussians directly from uncalibrated input images, eliminating the need for per-scene optimization [S1, unofficial]. It differs from iterative methods by using a transformer-based encoder-decoder to predict Gaussian parameters, depth, and camera poses in a single forward pass, leveraging pseudo-geometry priors from a pretrained Vision Transformer (VGGT) for supervision [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of $K$ uncalibrated RGB images $I \in \mathbb{R}^{1 \times K \times 3 \times 448 \times 448}$ [S1, unofficial].
*   **Outputs:** A set of 3D Gaussians $\mathcal{G}$ (defined by mean $\mu$, scale $s$, rotation $r$, opacity $\alpha$, and color $c$) and predicted camera poses (extrinsic $E$ and intrinsic $K$) [S1, unofficial].

**Data Structures:**
The method constructs pixel-wise 3D Gaussians from the decoder outputs. These are then voxelized into "pre-voxel 3D Gaussians" using a Differentiable Voxelization module to handle density and rendering efficiency [S1, unofficial].

**Algorithm Steps:**
1.  **Feature Encoding:** Input images are processed by a transformer-based geometry encoder to extract multi-scale features [S1, unofficial].
2.  **Parameter Prediction:** Three decoder heads ($F_G, F_D, F_C$) predict:
    *   $F_G$: Gaussian parameters ($\mu, \sigma, r, s, c$) [S1, unofficial]. Note: $\sigma$ typically denotes standard deviation related to scale, or the covariance components.
    *   $F_D$: A depth map $D$ [S1, unofficial].
    *   $F_C$: Camera poses $p$ (extrinsic and intrinsic matrices) [S1, unofficial].
3.  **Gaussian Construction:** The predicted parameters are used to construct a set of pixel-wise 3D Gaussians [S1, unofficial].
4.  **Differentiable Voxelization:** The pixel-wise Gaussians are voxelized into pre-voxel 3D Gaussians [S1, unofficial].
5.  **Rendering:** Multi-view images and depth maps are rendered from the voxelized Gaussians using differentiable rasterization [S1, unofficial].
6.  **Loss Computation:**
    *   **RGB Loss:** Rendered images are compared against ground truth images using an RGB loss (e.g., L1 or LPIPS) [S1, unofficial].
    *   **Geometry Loss:** Rendered depth maps, decoded depth $D$, and predicted poses $p$ are used to compute geometry losses [S1, unofficial].
7.  **Pseudo-Geometry Supervision:** The geometry predictions are supervised by pseudo-geometry priors obtained from a pretrained VGGT model [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def anysplat_pipeline(images):
    """
    images: torch.Tensor of shape [1, K, 3, 448, 448]
    Returns: gaussians (3D Gaussian parameters), pred_context_pose (camera poses)
    """
    # 1. Preprocess: Normalize images from [0, 1] to [-1, 1] if required by model, 
    # or [0, 1] if model expects [0, 1]. The code snippet shows (images+1)*0.5, 
    # implying input might be [-1, 1] normalized, converting to [0, 1].
    # However, standard image inputs are often [0, 1]. The snippet suggests:
    # images_input = (images + 1) * 0.5 
    # This implies the 'images' passed to inference are in [-1, 1] range.
    
    # 2. Forward Pass
    # The model consists of a geometry encoder and three decoder heads (F_G, F_D, F_C)
    gaussians, pred_context_pose = model.inference(images)
    
    # 3. Extract Poses
    pred_all_extrinsic = pred_context_pose['extrinsic']
    pred_all_intrinsic = pred_context_pose['intrinsic']
    
    # 4. Voxelization (Internal to model or separate module)
    # The model internally voxelizes the pixel-wise Gaussians into pre-voxel 3D Gaussians
    # using a Differentiable Voxelization module.
    
    # 5. Rendering and Loss (During Training)
    # Rendered images and depth maps are generated from the voxelized Gaussians.
    # Losses are computed against GT images and VGGT pseudo-geometry priors.
    
    return gaussians, pred_context_pose
```

**Parameters:**
*   Input resolution: $448 \times 448$ pixels [S1, unofficial].
*   Number of views $K$: Variable, typically small (e.g., 3-5) for unconstrained views [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Uncalibrated Inputs:** The method must handle unknown camera poses, relying on the network to predict them and the VGGT priors for geometric consistency [S1, unofficial].
*   **Voxelization Artifacts:** The Differentiable Voxelization step is critical for managing Gaussian density and preventing rendering artifacts [S1, unofficial].

## Subtopics

### [SubSplat](13524-subsplat.md)
A high-resolution variant using a Sub-pixel Gaussian Reparameterizer to increase density from low-resolution features.

### [Deformable Radial Kernel (DRK)](13699-deformable-radial-kernel-drk.md)
A variant replacing fixed Gaussians with flexible planar kernels for better geometric fidelity with fewer primitives.

## Operating characteristics
*   **Time Complexity:** Inference is $O(1)$ with respect to scene complexity, as it is a single forward pass through a neural network. Training time depends on the dataset size and VGGT prior computation.
*   **Memory Complexity:** Depends on the number of predicted Gaussians and the voxel grid resolution. The voxelization step helps manage memory by aggregating nearby Gaussians [S1, unofficial].
*   **Strengths:** Fast inference, handles uncalibrated views, generalizes to unseen scenes without per-scene optimization [S1, unofficial].
*   **Limitations:** Quality depends on the generalization capability of the network and the accuracy of VGGT pseudo-geometry priors. May struggle with highly dynamic or out-of-distribution scenes [S1, unofficial].
*   **Hardware:** Requires GPU for inference and training. PyTorch 2.2.0 and CUDA 12.1 recommended [S1, unofficial].

## Background
AnySplat is part of the family of feed-forward generalizable reconstruction methods for 3D Gaussian Splatting. It builds on the principles of mapping input features to Gaussian parameters without per-scene optimization [16746-feed-forward-generalizable-reconstruction.md]. It specifically addresses the challenge of unconstrained views by predicting camera poses and leveraging pretrained vision models for geometric supervision [S1, unofficial].

## Sources
[S1] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
