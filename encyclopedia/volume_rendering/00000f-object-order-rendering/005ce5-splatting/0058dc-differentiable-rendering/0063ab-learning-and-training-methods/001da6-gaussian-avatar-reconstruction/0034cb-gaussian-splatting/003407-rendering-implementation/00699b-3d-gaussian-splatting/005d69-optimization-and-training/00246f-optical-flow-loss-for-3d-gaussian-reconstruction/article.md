# Optical Flow Loss for 3D Gaussian Reconstruction

Also called: Flow-guided shape reconstruction, L2 flow loss, Optical Flow Weak Supervision, Optical flow regularization

## Overview
This method improves the 3D shape reconstruction of static objects from video sequences by incorporating an optical flow loss term into the optimization objective [S2, unofficial]. It leverages the temporal prior inherent in video data to resolve concavity/convexity ambiguities that are present when relying solely on silhouette and color loss terms [S2, unofficial]. Unlike standard photometric approaches, this technique uses optical flow as a local estimator of surface correspondence, which is robust to long-term lighting changes and autoexposure variations common in user-captured videos [S2, unofficial].

## How it works
The method extends the standard 3D Gaussian Splatting optimization loop by adding a flow consistency term [S2, unofficial]. It assumes that the 3D scene is static and that the camera poses for the input video sequence are known [S2, unofficial].

### Inputs and Outputs
*   **Inputs**: A sequence of video frames, corresponding camera poses (extrinsics), and a single inverse focal length parameter (intrinsics) [S2, unofficial]. The method assumes images lack distortion, projection is centered, and pixels are square [S2, unofficial].
*   **Outputs**: Optimized 3D Gaussian parameters (means, covariances, opacities, colors) that minimize the combined loss [S2, unofficial].

### Data Structures
The method operates on the standard 3D Gaussian Set [S2, unofficial]. For flow computation, it requires:
*   **Camera Poses**: A list of $4 \times 4$ transformation matrices $T_i$ for each frame $i$ [S2, unofficial].
*   **Inverse Focal Length**: A scalar $f^{-1}$ [S2, unofficial].
*   **Image Dimensions**: Width $W$ and Height $H$ [S2, unofficial].

### Numbered Steps
1.  **Ray Sampling**: Select a batch of rays $\mathcal{R}$ from the current frame $i$ [S2, unofficial]. This is identical to the parent method's ray sampling, using the camera intrinsics and extrinsics for frame $i$ [S2, unofficial].
2.  **Forward Rendering (Color)**: Render the color image $\hat{C}_i$ for frame $i$ using the current 3D Gaussians [S2, unofficial].
3.  **Forward Rendering (Depth/Coordinates)**: Render the depth image $D_i$ and the 3D coordinates of the hit points for each pixel [S2, unofficial]. The 3D coordinate $\mathbf{P}$ for a ray is obtained by back-projecting the depth $D$ from the camera center [S2, unofficial]. Specifically, if $\mathbf{r}_{dir}$ is the ray direction in camera space and $D$ is the depth, the point in camera space is $\mathbf{P}_{cam} = D \cdot \mathbf{r}_{dir}$ [S2, unofficial]. Transform this to world space using the camera pose $T_i$: $\mathbf{P}_{world} = T_i \cdot [D \cdot \mathbf{r}_{dir}, 1]^T$ [S2, unofficial].
4.  **Compute Rendered Flow ($\hat{f}$)**:
    *   For each ray in the batch, take the 3D world coordinate $\mathbf{P}_{world}$ obtained in Step 3 [S2, unofficial].
    *   Transform $\mathbf{P}_{world}$ into the coordinate system of the adjacent frame $i+1$ using $T_{i+1}^{-1}$: $\mathbf{P}_{cam\_next} = T_{i+1}^{-1} \cdot [\mathbf{P}_{world}, 1]^T$ [S2, unofficial].
    *   Project $\mathbf{P}_{cam\_next}$ onto the image plane of frame $i+1$ using the inverse focal length $f^{-1}$ and image dimensions [S2, unofficial]. Let the projected pixel coordinates be $(u_{next}, v_{next})$ [S2, unofficial].
    *   Similarly, transform $\mathbf{P}_{world}$ into the coordinate system of the previous frame $i-1$ using $T_{i-1}^{-1}$ and project to get $(u_{prev}, v_{prev})$ [S2, unofficial].
    *   The rendered forward flow estimate for the pixel is $\hat{f}_{forward} = (u_{next} - u_i, v_{next} - v_i)$ [S2, unofficial].
    *   The rendered backward flow estimate is $\hat{f}_{backward} = (u_{prev} - u_i, v_{prev} - v_i)$ [S2, unofficial].
    *   [G] If the $Z$ component of $\mathbf{P}_{cam\_next}$ is $\le 0$, the point is behind the camera and flow is undefined; these rays should be masked out or handled with a large loss penalty.
5.  **Compute Ground Truth Flow ($f$)**:
    *   Obtain ground truth optical flow maps between frame $i$ and $i+1$, and between $i$ and $i-1$, using an external optical flow estimator (e.g., Unimatch [70] or classic methods like [13]) [S2, unofficial].
    *   The ground truth flow $f$ is measured in pixels, scaled to half the shorter image dimension to normalize the loss magnitude [S2, unofficial].
6.  **Loss Computation**:
    *   Compute the accumulated opacity $\alpha$ for each ray from the volume rendering process [S2, unofficial].
    *   Compute Color Loss $L_C$:
        $$ L_C = \alpha \cdot \| c - \hat{c} \|_1 $$
        where $c$ is the ground truth color and $\hat{c}$ is the rendered color [S2, unofficial]. Colors should be gamma-corrected back to linear intensity before computation [S2, unofficial].
    *   Compute Flow Loss $L_F$:
        $$ L_F = \alpha \cdot \| f - \hat{f} \|_1 $$
        where $f$ is the ground truth flow and $\hat{f}$ is the rendered flow [S2, unofficial].
    *   Compute Base Loss $L_M$: This is the standard reconstruction loss (e.g., MSE or SSIM) from the parent method [S2, unofficial].
    *   Compute Total Loss $L$:
        $$ L = L_M + \lambda_C L_C + \lambda_F L_F $$
        where $\lambda_C$ and $\lambda_F$ are the loss weights [S2, unofficial].
7.  **Backpropagation**: Compute gradients of $L$ with respect to the Gaussian parameters and update them using the Adam optimizer [S2, unofficial].
8.  **Scheduling**: Apply learning rate decay based on statistical criteria [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_flow_loss(gaussians, camera_poses, current_frame_idx, 
                      gt_flow_forward, gt_flow_backward, 
                      inv_focal_length, image_width, image_height):
    """
    Computes the optical flow loss for a batch of rays.
    
    Args:
        gaussians: 3D Gaussian Set object with methods to render depth and 3D coords.
        camera_poses: List of 4x4 numpy arrays for all frames.
        current_frame_idx: Index of the current frame being rendered.
        gt_flow_forward: Ground truth flow from current_frame_idx to current_frame_idx + 1.
        gt_flow_backward: Ground truth flow from current_frame_idx to current_frame_idx - 1.
        inv_focal_length: Scalar inverse focal length.
        image_width, image_height: Image dimensions.
        
    Returns:
        flow_loss: Scalar loss value.
    """
    # 1. Render depth and 3D coordinates for current frame
    # Assume render_depth_and_coords returns:
    # depth_map: (H, W)
    # hit_points_world: (H, W, 3) - 3D world coordinates of the hit points
    # opacities: (H, W) - accumulated opacity per pixel
    depth_map, hit_points_world, opacities = gaussians.render_depth_and_coords(
        camera_poses[current_frame_idx], inv_focal_length, image_width, image_height
    )
    
    # 2. Get adjacent poses
    pose_next = camera_poses[current_frame_idx + 1]
    pose_prev = camera_poses[current_frame_idx - 1]
    
    # 3. Transform hit points to next and previous camera frames
    # hit_points_world is (H, W, 3). Convert to homogeneous (H, W, 4)
    hit_points_hom = np.concatenate([hit_points_world, np.ones((hit_points_world.shape[0], hit_points_world.shape[1], 1))], axis=-1)
    
    # Transform to next frame
    hit_points_next_cam_hom = np.linalg.inv(pose_next) @ hit_points_hom.transpose(2, 0, 1) # (4, H, W)
    hit_points_next_cam = hit_points_next_cam_hom[:3, :, :].transpose(1, 2, 0) # (H, W, 3)
    
    # Transform to previous frame
    hit_points_prev_cam_hom = np.linalg.inv(pose_prev) @ hit_points_hom.transpose(2, 0, 1)
    hit_points_prev_cam = hit_points_prev_cam_hom[:3, :, :].transpose(1, 2, 0)
    
    # 4. Project to image coordinates
    # Projection assumes pinhole model with centered principal point and square pixels [S2, unofficial]
    # u = (x_cam / z_cam) / inv_focal_length + width/2
    eps = 1e-6
    
    # Project to next frame
    u_next = (hit_points_next_cam[:, :, 0] / (hit_points_next_cam[:, :, 2] + eps)) / inv_focal_length + image_width / 2
    v_next = (hit_points_next_cam[:, :, 1] / (hit_points_next_cam[:, :, 2] + eps)) / inv_focal_length + image_height / 2
    
    # Project to previous frame
    u_prev = (hit_points_prev_cam[:, :, 0] / (hit_points_prev_cam[:, :, 2] + eps)) / inv_focal_length + image_width / 2
    v_prev = (hit_points_prev_cam[:, :, 1] / (hit_points_prev_cam[:, :, 2] + eps)) / inv_focal_length + image_height / 2
    
    # Current frame coordinates (center of pixel)
    u_curr = np.arange(image_width).reshape(1, -1) + 0.5
    v_curr = np.arange(image_height).reshape(-1, 1) + 0.5
    u_curr = np.tile(u_curr, (image_height, 1))
    v_curr = np.tile(v_curr, (1, image_width))
    
    # 5. Compute rendered flow
    # Forward flow: change from current to next
    f_hat_forward_u = u_next - u_curr
    f_hat_forward_v = v_next - v_curr
    
    # Backward flow: change from current to prev
    f_hat_backward_u = u_prev - u_curr
    f_hat_backward_v = v_prev - v_curr
    
    # 6. Mask invalid points (behind camera) [G]
    valid_next = hit_points_next_cam[:, :, 2] > eps
    valid_prev = hit_points_prev_cam[:, :, 2] > eps
    valid = valid_next & valid_prev
    
    # 7. Compute L1 loss with opacity weighting [S2, unofficial]
    # Ground truth flows are pre-scaled to half the shorter image dimension [S2, unofficial]
    loss_forward = np.abs(gt_flow_forward[:, :, 0] - f_hat_forward_u) + np.abs(gt_flow_forward[:, :, 1] - f_hat_forward_v)
    loss_backward = np.abs(gt_flow_backward[:, :, 0] - f_hat_backward_u) + np.abs(gt_flow_backward[:, :, 1] - f_hat_backward_v)
    
    # Apply opacity and validity mask
    weighted_loss_forward = opacities * loss_forward * valid
    weighted_loss_backward = opacities * loss_backward * valid
    
    # Mean loss over valid pixels
    num_valid = np.sum(valid)
    if num_valid == 0:
        return 0.0
        
    flow_loss = (np.sum(weighted_loss_forward) + np.sum(weighted_loss_backward)) / num_valid
    
    return flow_loss
```

### Parameters
*   $\lambda_C$ (Color Loss Weight): 4.5 [S2, unofficial].
*   $\lambda_F$ (Flow Loss Weight): 210 [S2, unofficial].
*   These values are fixed for all experiments in the source [S2, unofficial].
*   Flow is measured in the scale of half the shorter image dimension [S2, unofficial].

### Edge Cases and Pitfalls
*   **Shape Uncertainty Artifacts**: Without flow regularization, rendered flow estimates exhibit artifacts due to shape uncertainty [S2, unofficial].
*   **Color Fidelity**: Adding flow loss slightly hurts the color fidelity of the reconstructed models [S2, unofficial].
*   **Camera Intrinsics**: The method assumes a single inverse focal length parameter, no distortion, centered projection, and square pixels [S2, unofficial]. [G] Deviations from this may require modification of the projection step.
*   **Temporal Consistency**: The method requires adjacent frames [S2, unofficial]. [G] For the first and last frames in a sequence, one adjacent flow direction may be unavailable. [G] The source implies using both forward and backward flow for each ray, so boundary frames might need special handling (e.g., only using the available flow direction).

## Operating characteristics
*   **Time Complexity**: [G] The additional cost is the computation of rendered flow, which involves transforming 3D points and projecting them. [G] This is $O(N)$ where $N$ is the number of rays in the batch. The external optical flow estimation is performed offline or per-frame and is not part of the per-step optimization cost, though it is required for the loss computation [S2, unofficial].
*   **Memory Complexity**: No significant additional memory beyond storing the ground truth flow maps and intermediate 3D coordinates [S2, unofficial].
*   **Measured Performance**: On CO3D sequences, using Color & Learned Flow resulted in a Depth Error of 0.155 and a Runtime of 15 seconds, compared to 0.271 and 17 seconds for No Color or Flow [S2, unofficial].
*   **Strengths**: Resolves concavity/convexity ambiguities; robust to lighting changes; produces smoother depth maps; learned flow methods are real-time on GPU [S2, unofficial].
*   **Limitations**: Slightly reduces color fidelity; requires known camera poses; assumes static scene [S2, unofficial].
*   **When to Use**: When shape accuracy is more important than perfect color fidelity, and when video sequences with known poses are available [S2, unofficial].
*   **When Not to Use**: When color fidelity is the primary goal, or when camera poses are unknown [S2, unofficial].

## Background
This technique was introduced in "Flexible Techniques for Differentiable Rendering with 3D Gaussians" [S2, unofficial]. It builds on the observation that optical flow provides a useful signal for shape estimation, as it is a local estimator robust to long-term lighting changes [S2, unofficial]. The method draws inspiration from classic techniques for shape estimation and sparse particle video trackers [S2, unofficial]. It leverages state-of-the-art learned optical flow methods (e.g., Unimatch [70]) which provide better priors for texture-less areas compared to classic optical flow [13] [S2, unofficial].

## Sources
[S1] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S2] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
