# RoGUENeRF

Also called: RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF

## Overview
RoGUENeRF is a post-processing enhancer for trained Neural Radiance Fields (NeRF) that improves visual fidelity and restores high-frequency textures in rendered images while maintaining view-consistency and robustness to inaccurate camera calibration [S1]. Unlike methods that require extensive re-optimization or large-scale synthetic pre-training datasets, RoGUENeRF operates as a universal enhancer that can be fine-tuned on small real-world datasets, leveraging the geometry already learned by the NeRF to align and fuse features from neighboring training views [S1]. It sets itself apart from siblings like [Aug-NeRF](10294-aug-nerf.md) (which modifies the loss landscape) and [NeRFLiX](10294-aug-nerf.md) (which requires massive synthetic pre-training) by using a lightweight, geometry-aware alignment and fusion pipeline that runs in under 60 minutes of fine-tuning and offers significantly faster inference speeds [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A trained NeRF model, the set of training images $\{H_i\}_{i=1}^M$ with known camera poses $\{C_i\}_{i=1}^M$, and a novel test camera pose $C_k$.
*   **Outputs:** An enhanced RGB image $I_k^{enhanced}$ corresponding to the novel view.

**Data Structures and Layouts:**
The method processes features through specific network blocks. The layouts are defined as follows:

1.  **Convolutional Encoder Block:**
    *   **Input:** Concatenated tensor of shape $[H, W, C_{in}]$ where $C_{in}$ comprises:
        *   Neighbour Features (RGB or extracted features)
        *   Test Features (Rendered NeRF features)
        *   Neighbour Depth ($D_i$)
        *   Test Depth ($D_k$)
        *   Neighbour Camera Pose ($C_i$)
        *   Test Camera Pose ($C_k$)
    *   **Layer Sequence:**
        1.  `conv2d` (kernel=3, channels=64, stride=1, padding=1), Leaky ReLU (slope=0.01)
        2.  `conv2d` (kernel=3, channels=64, stride=1, padding=1), Leaky ReLU (slope=0.01)
        3.  `linear` (in=12, out=128), Leaky ReLU (slope=0.01) *[Note: Input dimension 12 likely refers to flattened pose/depth metadata or specific channel aggregation, see [S1] Fig 2]*
        4.  `linear` (in=128, out=1), Sigmoid
        5.  `conv2d` (kernel=3, channels=128, stride=1, padding=1), Leaky ReLU (slope=0.01)
        6.  `conv2d` (kernel=3, channels=64, stride=1, padding=1), Sigmoid
        7.  `conv2d` (kernel=1, channels=64, stride=1, padding=0)
    *   **Outputs:**
        *   Image Features
        *   Spatial Attention Mask
        *   Camera Attention Mask

2.  **Non-Rigid Refinement Network:**
    *   **Input:** Concatenated tensor of Neighbour Features and Test Features.
    *   **Layer Sequence:**
        1.  `conv2d` (kernel=7, channels=64, stride=4, padding=0), ReLU
        2.  `conv2d` (kernel=3, channels=64, stride=1, padding=1), ReLU
        3.  `conv2d` (kernel=3, channels=64, stride=1, padding=1) + ReLU (Residual connection)
        4.  `conv2d` (kernel=3, channels=64, stride=2, padding=1), ReLU
        5.  `conv2d` (kernel=3, channels=64, stride=2, padding=1) + ReLU (Residual connection)
        6.  `conv2d` (kernel=3, channels=64, stride=1, padding=1), ReLU
        7.  `conv2d` (kernel=3, channels=64, stride=1, padding=1)
        8.  `conv2d` (kernel=3, channels=128, stride=1, padding=1), ReLU
        9.  `conv2d` (kernel=3, channels=64, stride=1, padding=1)
        10. `conv2d` (kernel=3, channels=2, stride=1, padding=1) *[Output: Optical Flow]*
    *   **Process:** The network iterates 32 times. The output flow is upsampled bilinearly to full resolution.

**Numbered Steps:**

1.  **Nearest Neighbour Selection:**
    Given a novel test camera pose $C_k = (R_k, t_k)$ and training poses $\{C_i = (R_i, t_i)\}_{i=1}^M$, identify the $N$ training images with the largest content overlap.
    *   Compute angular distance between rotations:
        $$dist^{i}_{ang}(R_{k}, R_{i}) = \frac {1}{3} \lvert \pi (R_{k}) - \pi (R_{i}) \rvert _{1}$$
        where $\pi(R_i)$ are the Euler angles of rotation matrix $R_i$, and $\lvert \cdot \rvert_1$ is the L1 norm [S1].
    *   Compute positional distance:
        $$dist^{i}_{pos}(t_{k}, t_{i}) = \frac {1}{3} \lvert t_{k} - t_{i} \rvert _{1}$$
    *   Select the top $N$ neighbors (empirically $N=5$) based primarily on $dist^{i}_{pos}$, followed by $dist^{i}_{ang}$ [S1].

2.  **Feature Extraction:**
    Extract features from the selected Neighbour Images and the Rendered Test Image using the Convolutional Encoder Block defined above. This produces Image Features, Spatial Attention Mask, and Camera Attention Mask [S1].

3.  **3D Alignment:**
    Align the Neighbour Features to the novel test view $C_k$.
    *   Use the NeRF-rendered depth maps $D_i$ (neighbour) and $D_k$ (test) along with camera poses $C_i$ and $C_k$.
    *   Perform 3D reprojection using a geometry remap function (e.g., Kornia) with bilinear interpolation and zero padding [S1]. This warps the neighbour features into the coordinate frame of the test view.

4.  **Non-Rigid Refinement:**
    Compensate for geometric inaccuracies and misalignments.
    *   Input the aligned Neighbour Features and Test Features into the Non-Rigid Refinement Network.
    *   The network estimates optical flow at $8\times$ downsampled resolution.
    *   Iterate the refinement process 32 times [S1].
    *   Bilinearly upsample the estimated flow field to full resolution.
    *   Warp the aligned Neighbour Features using the refined flow field via a grid sampling function (e.g., PyTorch `grid_sample`) with bilinear interpolation to produce Refined Features [S1].

5.  **Geometry-Aware Attention and Fusion:**
    *   Combine the Refined Features with the Test Features.
    *   Apply the Spatial Attention Mask and Camera Attention Mask generated in Step 2 to regulate the contribution of misaligned regions and modulate feature importance based on camera geometry [S1].
    *   Fuse the regulated features.

6.  **Uformer Enhancement:**
    *   Pass the fused features through a Uformer network.
    *   The Uformer input channels are modified to accept 64-channel features (instead of the original 3-channel RGB) [S1].
    *   Perform a single inference pass without Test-Time Augmentation (TTA) [S1].

**Parameters:**
*   **Leaky ReLU Slope:** 0.01 [S1].
*   **Weight Initialization:** He initialization for learnable weights [S1].
*   **Convolutional Padding:** Zero padding for all convolutional layers [S1].
*   **Camera Pose Representation:** Rotations are converted to Euler angles before processing by the attention module [S1].
*   **Uformer Input Channels:** 64 [S1].
*   **Optical Flow Resolution:** Estimated at $8\times$ downsampled resolution, upsampled to full resolution [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class RoGUENeRF:
    def __init__(self, nerf_model, training_data, neighbor_count=5):
        """
        Initialize RoGUENeRF.
        :param nerf_model: Trained NeRF model capable of rendering RGB and Depth.
        :param training_data: Dict containing training images H, poses C.
        :param neighbor_count: Number of nearest neighbors to select.
        """
        self.nerf = nerf_model
        self.training_images = training_data['images'] # List of H_i
        self.training_poses = training_data['poses']   # List of C_i = (R_i, t_i)
        self.neighbor_count = neighbor_count
        
        # Initialize sub-networks
        self.encoder = ConvolutionalEncoderBlock()
        self.refinement_net = NonRigidRefinementNetwork()
        self.uformer = Uformer(input_channels=64)
        
        # Load pre-trained weights for sub-networks if available
        # self.load_weights()

    def compute_pose_distance(self, pose_k, pose_i):
        """
        Compute angular and positional distance between two camera poses.
        :param pose_k: Tuple (R_k, t_k)
        :param pose_i: Tuple (R_i, t_i)
        :return: Tuple (dist_ang, dist_pos)
        """
        R_k, t_k = pose_k
        R_i, t_i = pose_i
        
        # Convert rotation matrices to Euler angles
        euler_k = self.matrix_to_euler(R_k)
        euler_i = self.matrix_to_euler(R_i)
        
        # Angular distance: Mean L1 norm of Euler angle differences
        diff_ang = euler_k - euler_i
        dist_ang = torch.mean(torch.abs(diff_ang))
        
        # Positional distance: Mean L1 norm of translation differences
        diff_pos = t_k - t_i
        dist_pos = torch.mean(torch.abs(diff_pos))
        
        return dist_ang, dist_pos

    def select_neighbors(self, test_pose):
        """
        Select nearest neighbor training images based on pose distance.
        """
        distances = []
        for i, pose_i in enumerate(self.training_poses):
            dist_ang, dist_pos = self.compute_pose_distance(test_pose, pose_i)
            # Primary sort key is positional distance, secondary is angular
            distances.append((dist_pos, dist_ang, i))
        
        # Sort by positional distance
        distances.sort(key=lambda x: x[0])
        
        # Select top N indices
        neighbor_indices = [idx for _, _, idx in distances[:self.neighbor_count]]
        return neighbor_indices

    def extract_features(self, neighbor_images, test_image, neighbor_depths, test_depth, 
                         neighbor_poses, test_pose):
        """
        Extract features using the Convolutional Encoder Block.
        """
        # Prepare input tensor for encoder
        # Concatenate: Neighbour Features, Test Features, Neighbour Depth, Test Depth, 
        # Neighbour Cam Pose, Test Cam Pose
        # Note: In implementation, images/depths are likely processed into feature maps first
        # or the encoder handles raw inputs. Based on Fig 2, inputs are features/depths/poses.
        
        # Assuming 'features' are extracted from RGB images via a standard encoder or the 
        # first conv layers of the encoder itself. 
        # For clarity, we assume the encoder takes the concatenated tensor described in layout.
        
        # Placeholder for actual tensor construction logic which depends on specific 
        # feature extraction pipeline not fully detailed in text beyond "extract features using... block"
        # We assume the block processes the concatenated inputs directly.
        
        input_tensor = self.prepare_encoder_input(
            neighbor_images, test_image, 
            neighbor_depths, test_depth,
            neighbor_poses, test_pose
        )
        
        image_features, spatial_mask, camera_mask = self.encoder(input_tensor)
        return image_features, spatial_mask, camera_mask

    def refine_features(self, aligned_neighbor_features, test_features):
        """
        Perform non-rigid refinement using optical flow.
        """
        # Input: Aligned Neighbour Features, Test Features
        # Estimate flow at 8x downsampled resolution
        flow = self.refinement_net(aligned_neighbor_features, test_features)
        
        # Iterate 32 times (simplified representation of iterative refinement)
        for _ in range(32):
            # Upsample flow to full resolution
            flow_full = F.interpolate(flow, scale_factor=8, mode='bilinear', align_corners=False)
            
            # Warp neighbour features using flow
            # grid_sample expects flow in shape (N, H, W, 2)
            aligned_neighbor_features = self.warp_features(aligned_neighbor_features, flow_full)
            
            # Re-estimate flow on warped features (iterative step)
            flow = self.refinement_net(aligned_neighbor_features, test_features)
            
        return aligned_neighbor_features

    def warp_features(self, features, flow):
        """
        Warp features using grid_sample with bilinear interpolation.
        """
        # Grid sample requires normalized coordinates (-1 to 1)
        # Assuming flow is in pixel coordinates, it needs normalization
        h, w = features.shape[2], features.shape[3]
        grid = self.flow_to_grid(flow, h, w)
        warped = F.grid_sample(features, grid, mode='bilinear', padding_mode='zeros')
        return warped

    def flow_to_grid(self, flow, h, w):
        """
        Convert optical flow to grid for grid_sample.
        """
        # Standard conversion logic
        # ... implementation details ...
        pass

    def enhance(self, test_pose):
        """
        Main inference pipeline.
        """
        # 1. Render NeRF output for test pose
        test_rgb, test_depth = self.nerf.render(test_pose)
        
        # 2. Select Neighbors
        neighbor_indices = self.select_neighbors(test_pose)
        neighbor_images = [self.training_images[i] for i in neighbor_indices]
        neighbor_poses = [self.training_poses[i] for i in neighbor_indices]
        
        # 3. Render NeRF outputs for neighbors (for depth alignment)
        neighbor_depths = []
        for pose in neighbor_poses:
            _, depth = self.nerf.render(pose)
            neighbor_depths.append(depth)
            
        # 4. Extract Features
        # Note: Neighbor images are ground truth H_i, not NeRF rendered
        image_features, spatial_mask, camera_mask = self.extract_features(
            neighbor_images, test_rgb, 
            neighbor_depths, test_depth,
            neighbor_poses, test_pose
        )
        
        # 5. 3D Alignment
        # Align neighbor features to test view using depth and poses
        aligned_features = self.align_3d(neighbor_images, neighbor_depths, neighbor_poses, 
                                         test_pose, test_depth)
        
        # 6. Non-Rigid Refinement
        refined_features = self.refine_features(aligned_features, image_features)
        
        # 7. Geometry-Aware Attention
        # Apply masks to regulate features
        regulated_features = self.apply_attention(refined_features, image_features, 
                                                  spatial_mask, camera_mask)
        
        # 8. Uformer Enhancement
        enhanced_output = self.uformer(regulated_features)
        
        return enhanced_output

    def align_3d(self, neighbor_images, neighbor_depths, neighbor_poses, test_pose, test_depth):
        """
        Perform 3D alignment using Kornia geometry remap.
        """
        aligned_list = []
        for img, depth, pose in zip(neighbor_images, neighbor_depths, neighbor_poses):
            # Use Kornia remap function with bilinear interpolation and zero padding
            aligned = self.remap_with_kornia(img, depth, pose, test_pose, test_depth)
            aligned_list.append(aligned)
        return torch.stack(aligned_list)

    def remap_with_kornia(self, img, depth, src_pose, dst_pose, dst_depth):
        """
        Remap image from source pose to destination pose using depth.
        """
        # Implementation using Kornia.geometry.remap
        # ...
        pass

    def apply_attention(self, features, test_features, spatial_mask, camera_mask):
        """
        Fuse features using attention masks.
        """
        # Combine masks
        combined_mask = spatial_mask + camera_mask
        # Apply mask to features
        regulated = features * combined_mask
        # Fuse with test features
        # ... specific fusion logic ...
        return regulated

    def matrix_to_euler(self, R):
        """
        Convert rotation matrix to Euler angles.
        """
        # Standard conversion
        pass
```

## Subtopics

### 3D Alignment
Aligns training features to the test view using NeRF-rendered depth and camera poses. Sets itself apart by using explicit geometry for initial alignment, reducing the burden on the refinement network. Choose when accurate depth maps are available from the NeRF. Link: [3D Alignment](10294-aug-nerf.md) (conceptual sibling in geometry-aware methods).

### Non-Rigid Refinement
Uses an iterative optical flow network to correct misalignments caused by geometry estimation errors or non-rigid deformations. Sets itself apart by operating at a lower resolution ($8\times$) for efficiency and iterating to refine the flow. Choose when camera poses are slightly inaccurate or scene has minor non-rigid elements. Link: [Optical Flow Refinement](10294-aug-nerf.md).

### Geometry-Aware Attention
Modulates feature fusion using spatial and camera attention masks. Sets itself apart by explicitly using camera pose (Euler angles) and spatial cues to weigh the contribution of neighboring features. Choose when view-consistency is critical. Link: [Attention Mechanisms](10294-aug-nerf.md).

## Operating characteristics
*   **Time Complexity:** The method involves feature extraction, 3D alignment, iterative optical flow estimation (32 iterations), and Uformer inference. The optical flow estimation is performed at $8\times$ downsampled resolution, reducing computational cost.
*   **Memory Complexity:** Requires storing features for neighboring views and the Uformer model. The Uformer is modified to accept 64-channel inputs.
*   **Measured Performance:**
    *   On NVIDIA V100 GPU at full float32 precision:
        *   512x512 resolution: 337ms [S1].
        *   1024x1024 resolution: 1346ms [S1].
    *   Comparison: Approximately 13x faster than NeRFLiX with TTA, 3x faster than NeRFLiX without TTA, and 3x slower than NeRFLiX++ [S1].
*   **Training/Fine-tuning Time:** Fine-tuning takes less than 60 minutes [S1].
*   **Data Efficiency:** Can be pre-trained and fine-tuned on small real-world datasets (e.g., LLFF). NeRFLiX requires orders of magnitude more data (Vimeo90K) to achieve similar improvements; on small datasets, NeRFLiX can degrade performance (e.g., -0.6dB PSNR on LLFF vs +0.70dB for RoGUENeRF) [S1].
*   **Strengths:** Robust to camera calibration errors, maintains view-consistency, restores high-frequency textures, fast inference, data-efficient.
*   **Limitations:** Relies on the quality of the initial NeRF geometry (depth maps). If the NeRF geometry is severely incorrect, alignment may fail. Requires a trained NeRF model as input.
*   **When to use:** When you have a trained NeRF with degraded high-frequency details and want to enhance it without re-training the NeRF, especially with limited data or noisy camera poses.
*   **When not to use:** When the initial NeRF geometry is highly inaccurate, or when real-time inference is critical and NeRFLiX++ speed is required (though RoGUENeRF is still fast).

## Background
RoGUENeRF was introduced by S. Catley-Chandar et al. in 2024 [S1]. It addresses the limitations of existing NeRF enhancement methods like NeRFLiX, which require large-scale synthetic pre-training datasets and are computationally expensive. RoGUENeRF leverages the geometry-consistent nature of NeRFs to align and fuse features from neighboring views, offering a robust and efficient alternative for enhancing NeRF renders in the wild.

## Sources
[S1] RoGUENeRF: A Robust Geometry-Consistent Universal Enhancer for NeRF, https://doi.org/10.1007/978-3-031-73254-6_4, fetched 2026-10-29
