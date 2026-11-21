# Multi-plane Neural Rendering

Also called: 3D Convolutional Renderer, MPP Network, Multi-Plane Projection, Neural Volume Rendering Network, NPR-MPP, Neural Point Cloud Rendering via Multi-Plane Projection

## Overview
Multi-plane Neural Rendering is a deep learning pipeline that renders images from raw 3D point clouds by projecting features into a layered 3D volume within the camera frustum, rather than directly onto a 2D image plane [S1]. Unlike previous methods that project features directly to 2D and suffer from visibility verification problems and noise interference [S1], this method maintains all features from points in the field-of-view, allowing a 3D CNN to automatically learn visibility and handle occlusions [S1]. It produces more stable renderings, particularly near object boundaries, and is robust to noisy and relatively sparse point clouds [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Raw 3D point cloud of a scene. Each point has a position and an appearance feature [S1].
*   **Output:** Image or image sequences from a novel view or along a novel camera trajectory [S1].

**Data Structures and Layout**
1.  **Point Feature:** A concatenation of an 8-dimensional learnable appearance feature and a normalized view direction vector (between camera position and point) [S1]. The total point feature dimension is 11 [S1].
2.  **3D Feature Volume:** Dimensions $P \times H \times W$, containing aggregated point features [S1]. $H \times W$ are the image dimensions, and $P$ is the pre-defined number of depth planes [S1].
3.  **Network Input:** Concatenation of the 3D feature volume and the normalized view direction [S1].
4.  **Network Output:** A tensor with 4 channels (RGB + blend weight) per plane [S1].

**Parameters**
*   $H, W$: Image dimensions. Typical values: $(480, 640)$ for ScanNet, $(512, 640)$ for Matterport 3D [S1].
*   $P$: Number of depth planes. Typical value: 32 [S1].
*   Point feature dimension: 11 (8 trainable appearance + 3 view direction) [S1].
*   Initialization: Point features initialized as 0.5 (5 dimensions) + RGB (3 dimensions) + viewpoint direction (3 dimensions) [S1]. Only the first 8 dimensions are updated [S1].
*   `use_dilation`: Boolean flag for dilated convolution in Conv6, default True [S2, unofficial].
*   Number of samples $d$: Default 32 [S2, unofficial].

**Steps**

1.  **Feature Initialization:** For each 3D point, initialize an 11-dimensional feature vector. The first 8 dimensions are learnable appearance features (initialized to 0.5), and the last 3 dimensions are the normalized view direction from the camera to the point [S1].
2.  **Frustum Definition:** Using the camera projection matrix, lift each pixel to 3D space to form a frustum defined by the minimum and maximum depth of the projected point cloud [S1].
3.  **Voxelization:** Divide the frustum into voxels with respect to image dimensions $H \times W$ and $P$ depth planes along the z-axis, resulting in $P \times H \times W$ frustum voxels [S1].
4.  **Feature Aggregation:** Aggregate features of points inside each frustum voxel using geometric rules. Points closer to the camera or pixel center are assigned larger weights [S1].
5.  **Network Input Preparation:** Concatenate the aggregated 3D feature volume ($P \times H \times W$) with the normalized view direction vector [S1].
6.  **3D CNN Encoding:** Pass the input through a U-Net-like 3D CNN encoder:
    *   Conv0: 1x1 kernel, 16 filters, LReLU [S2, unofficial].
    *   Conv1: 3x3 kernel, 32 filters, LReLU [S2, unofficial].
    *   Pool1: 2x2x2, stride 2 [S2, unofficial].
    *   Conv2: 3x3 kernel, 32 filters, LReLU [S2, unofficial].
    *   Pool2: 2x1x1, stride [2,1,1] [S2, unofficial].
    *   Conv3: 3x3 kernel, 64 filters, LReLU [S2, unofficial].
    *   Pool3: 2x2x2, stride 2 [S2, unofficial].
    *   Conv4: 3x3 kernel, 64 filters, LReLU [S2, unofficial].
    *   Pool4: 2x1x1, stride [2,1,1] [S2, unofficial].
    *   Conv5: 3x3 kernel, 128 filters, LReLU [S2, unofficial].
    *   Pool5: 2x2x2, stride 2 [S2, unofficial].
7.  **Bottleneck:** Conv6: 1x3x3 kernel, 128 filters, LReLU. If `use_dilation` is True, use dilation rate [1,2,2]; otherwise [1,1,1] [S2, unofficial].
8.  **3D CNN Decoding (U-Net Skip Connections):**
    *   Up1: Upsample Conv6 (2x2x2) and concatenate with Conv5 (128 channels) [S2, unofficial].
    *   Conv7: Process concatenated features [S2, unofficial].
    *   Up2: Upsample Conv7 (2x1x1) and concatenate with Conv4 (64 channels) [S2, unofficial].
    *   Conv8: Process concatenated features [S2, unofficial].
    *   Up3: Upsample Conv8 (2x2x2) and concatenate with Conv3 (64 channels) [S2, unofficial].
    *   Conv9: Process concatenated features [S2, unofficial].
    *   Up4: Upsample Conv9 (2x1x1) and concatenate with Conv2 (32 channels) [S2, unofficial].
    *   Conv10: Process concatenated features [S2, unofficial].
    *   Up5: Upsample Conv10 (2x2x2) and concatenate with Conv1 (32 channels) [S2, unofficial].
    *   Conv11: Process concatenated features [S2, unofficial].
9.  **Output Prediction:**
    *   Color: Conv12 (1x1x1, 3 filters, linear) predicts color $c_i$ [S2, unofficial].
    *   Density: Conv12_1 (1x1x1, 1 filter, linear) predicts density $\rho_i$ [S2, unofficial].
10. **Volume Rendering:** Accumulate color using discrete volume rendering formula over $d$ samples (default 32) [S2, unofficial]:
    $$ C = \frac{1}{N} \text{ or } \sum T_i \times (1 - \exp(-\rho_i \times \delta_i)) \times c_i $$
11. **Multi-plane Blending:** Alternatively, the network predicts RGB images $I_p$ and blending weights $\alpha_p$ for each plane $p$. The final output image $I$ is obtained by blending:
    $$ I = \sum_p I_p \cdot \alpha_p $$
    [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn.functional as F

def conv3d(x, out_channels, kernel_size, stride=1, padding=0, dilation=1):
    return F.conv3d(x, weight=None, bias=None, stride=stride, padding=padding, dilation=dilation)

def pool3d(x, kernel_size, stride=None):
    return F.max_pool3d(x, kernel_size=kernel_size, stride=stride if stride else kernel_size)

def upsample3d(x, scale_factor):
    return F.interpolate(x, scale_factor=scale_factor, mode='trilinear', align_corners=False)

def concat(a, b, dim=-1):
    return torch.cat([a, b], dim=dim)

def linear(x, out_channels):
    return F.linear(x.flatten(1), weight=None, bias=None).reshape(x.shape[:-1] + (out_channels,))

def multi_plane_neural_rendering(point_cloud, camera_intrinsics, camera_extrinsics, H, W, P):
    """
    Renders an image from a point cloud using multi-plane neural rendering.
    
    Args:
        point_cloud: Array of shape (N, 3) containing 3D coordinates.
        camera_intrinsics: 3x3 camera intrinsic matrix.
        camera_extrinsics: 4x4 camera extrinsic matrix.
        H: Height of the output image.
        W: Width of the output image.
        P: Number of depth planes.
        
    Returns:
        rendered_image: Array of shape (H, W, 3).
    """
    device = point_cloud.device
    num_points = point_cloud.shape[0]
    
    # Step 1: Initialize Point Features
    point_features = torch.zeros((num_points, 11), device=device)
    point_features[:, :8] = 0.5
    
    # Step 2: Compute View Directions
    points_homogeneous = torch.cat([point_cloud, torch.ones((num_points, 1), device=device)], dim=1)
    points_cam = (camera_extrinsics @ points_homogeneous.T).T[:, :3]
    norms = torch.norm(points_cam, dim=1, keepdim=True) + 1e-8
    view_directions = points_cam / norms
    point_features[:, 8:] = view_directions
    
    # Step 3: Define Frustum and Voxelize
    points_cam_homogeneous = torch.cat([points_cam, torch.ones((num_points, 1), device=device)], dim=1)
    points_img = (camera_intrinsics @ points_cam_homogeneous[:, :3].T).T
    points_img[:, :2] /= points_img[:, 2:3]
    
    valid_mask = (points_img[:, 0] >= 0) & (points_img[:, 0] < W) & \
                 (points_img[:, 1] >= 0) & (points_img[:, 1] < H) & \
                 (points_img[:, 2] > 0)
    
    valid_points = points_cam[valid_mask]
    valid_features = point_features[valid_mask]
    valid_u = points_img[valid_mask, 0]
    valid_v = points_img[valid_mask, 1]
    valid_z = valid_points[:, 2]
    
    min_depth = torch.min(valid_z)
    max_depth = torch.max(valid_z)
    depth_range = max_depth - min_depth + 1e-8
    
    voxel_grid = torch.zeros((P, H, W, 11), device=device)
    
    # Step 4: Feature Aggregation
    for i in range(len(valid_points)):
        u = int(torch.clamp(valid_u[i], 0, W-1).item())
        v = int(torch.clamp(valid_v[i], 0, H-1).item())
        p = int(torch.clamp((valid_z[i] - min_depth) / depth_range * P, 0, P-1).item())
        
        dist_cam = torch.norm(valid_points[i])
        weight = 1.0 / (dist_cam + 1e-8)
        
        voxel_grid[p, v, u] += valid_features[i] * weight
        
    # Step 5: Network Input Preparation
    view_dir_expanded = view_directions[0].unsqueeze(0).unsqueeze(0).unsqueeze(0)
    network_input = concat(voxel_grid.permute(0, 3, 1, 2).unsqueeze(0), 
                           view_dir_expanded.expand(P, 3, H, W).unsqueeze(0), dim=1)
    
    # Step 6-9: 3D CNN Forward Pass
    x = conv3d(network_input, 16, 1) # Conv0
    x = F.leaky_relu(x)
    x = conv3d(x, 32, 3) # Conv1
    x = F.leaky_relu(x)
    x = pool3d(x, 2) # Pool1
    x = conv3d(x, 32, 3) # Conv2
    x = F.leaky_relu(x)
    x = pool3d(x, (2,1,1), (2,1,1)) # Pool2
    conv2_out = x
    x = conv3d(x, 64, 3) # Conv3
    x = F.leaky_relu(x)
    x = pool3d(x, 2) # Pool3
    x = conv3d(x, 64, 3) # Conv4
    x = F.leaky_relu(x)
    x = pool3d(x, (2,1,1), (2,1,1)) # Pool4
    conv4_out = x
    x = conv3d(x, 128, 3) # Conv5
    x = F.leaky_relu(x)
    x = pool3d(x, 2) # Pool5
    conv5_out = x
    
    # Step 7: Bottleneck
    dilation = (1,2,2) if True else (1,1,1)
    x = conv3d(x, 128, (1,3,3), dilation=dilation) # Conv6
    x = F.leaky_relu(x)
    
    # Step 8: Decoder
    x = upsample3d(x, 2) # Up1
    x = concat(x, conv5_out, dim=1)
    x = conv3d(x, 128, 3) # Conv7
    x = F.leaky_relu(x)
    
    x = upsample3d(x, (2,1,1)) # Up2
    x = concat(x, conv4_out, dim=1)
    x = conv3d(x, 64, 3) # Conv8
    x = F.leaky_relu(x)
    
    x = upsample3d(x, 2) # Up3
    x = concat(x, conv3_out, dim=1)
    x = conv3d(x, 64, 3) # Conv9
    x = F.leaky_relu(x)
    
    x = upsample3d(x, (2,1,1)) # Up4
    x = concat(x, conv2_out, dim=1)
    x = conv3d(x, 32, 3) # Conv10
    x = F.leaky_relu(x)
    
    x = upsample3d(x, 2) # Up5
    x = concat(x, conv1_out, dim=1)
    x = conv3d(x, 32, 3) # Conv11
    x = F.leaky_relu(x)
    
    # Step 9: Output Prediction
    color = conv3d(x, 3, 1) # Conv12
    density = conv3d(x, 1, 1) # Conv12_1
    
    # Step 10 & 11: Rendering / Blending
    # S1 predicts blending weights instead of density for final output
    blend_weights = F.softmax(color[:, 3:4, :, :], dim=1)
    rgb_images = color[:, :3, :, :]
    
    rendered_image = torch.sum(rgb_images * blend_weights, dim=1)
    return rendered_image.permute(1, 2, 0)
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1]. Training takes on average 41.5 hours per scene on a GeForce 1080 Ti [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** Evaluated on ScanNet and Matterport 3D datasets [S1]. Produces more stable renderings compared to previous methods, especially near object boundaries [S1]. Performs reasonably well given relatively sparse point clouds [S1].
*   **Scaling:** Uses point clouds which are more scalable for larger scenes compared to DeepVoxel which relies on 3D volume [S1].
*   **Strengths:** Robust to noisy and relatively sparse point clouds [S1]. Handles occlusions and noises appropriately by projecting to a layered 3D volume [S1]. Temporal coherence in image sequences [S1].
*   **Limitations:** Requires a point cloud as input [S1]. Training time is significant [S1].
*   **When to use:** When rendering from raw 3D point clouds, especially in the presence of noise or sparsity, and when temporal stability is required [S1].
*   **When not to use:** When a dense 3D volume representation is already available and point clouds are not the primary input [S1].

## Background
Multi-plane Neural Rendering was proposed by Peng Dai, Yinda Zhang, Zhuwen Li, Shuaicheng Liu, and Bing Zeng from the University of Electronic Science and Technology of China, Google Research, and Nuro Inc [S1]. It was published in CVPR 2020 [S1]. The method addresses the limitations of previous neural point-based graphics approaches like NPG, which suffer from visibility verification problems and noise interference due to direct 3D-to-2D projection [S1]. It builds on the idea of using learnable point features and view-dependent appearance [S1].

## Sources
[S1] Neural Point Cloud Rendering via Multi-Plane Projection, https://doi.org/10.1109/cvpr42600.2020.00785, fetched 2026-10-30
[S2] https://github.com/Mayxmu/PCMP-NeRF, https://github.com/Mayxmu/PCMP_NeRF, fetched 2026-10-30 (unofficial)
