# SparSplat

Also called: Generalizable 2D Gaussian Splatting, SparSplat Model, SparSplat: Fast Multi-View Reconstruction with Generalizable 2D Gaussian Splatting

## Overview
SparSplat is a generalizable, feed-forward model that maps a sparse set of input images to 2D Gaussian Splatting (2DGS) parameters in a single forward pass to perform both 3D shape reconstruction and novel view synthesis [S1, unofficial]. It differs from its sibling [MVSGaussian](16571-mvsgaussian.md) by using 2D planar primitives instead of 3D volumetric primitives, which ensures multi-view depth consistency required for coherent TSDF mesh fusion [S1, unofficial]. Unlike prior generalizable methods designed primarily for novel view synthesis, SparSplat is explicitly optimized for feed-forward surface reconstruction, offering inference speeds nearly two orders of magnitude faster than implicit volume rendering methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $N$ color images $I_i \in \mathbb{R}^{H \times W \times 3}$ and their corresponding camera poses $P_i$ (rotation and translation). During training, $N=3$ and resolution is $640 \times 512$ [S1, unofficial].
*   **Outputs:** For a target view with pose $P_t$, the model $\Phi$ predicts a pixel-aligned 2DGS parameter map $O_t \in \mathbb{R}^{H \times W \times (2+4+3+1+3)}$ [S1, unofficial]. The channels correspond to:
    *   Scale $s$: 2 values (planar dimensions).
    *   Rotation $q$: 4 values (quaternion).
    *   Position $p$: 3 values (3D centroid).
    *   Opacity $\alpha$: 1 value.
    *   Color $c$: 3 values (RGB).

**Data Structures:**
The core data structure is the parameter map $O_t$. The position $p$ is derived from depth estimation, while $s, q, \alpha, c$ are predicted directly by the network.

**Algorithm Steps:**
1.  **Feature Encoding:**
    *   Extract monocular 2D semantic features from each input image $I_i$ using DINOv2 [S1, unofficial].
    *   Extract pairwise dense correspondence features from input images using MASt3R [S1, unofficial].
    *   An FPN network predicts a 2D feature map $f_i$ for each input image $I_i$ [S1, unofficial].
2.  **Feature Warping:**
    *   Warp the feature maps $f_i$ to the target view coordinate system using homography based on relative poses $P_t / P_i$, resulting in warped features $f_{t,i}$ [S1, unofficial].
3.  **Cost Volume and Depth Estimation (deepMVS branch):**
    *   Merge warped features $f_{t,i}$ into a cost volume [S1, unofficial].
    *   Process the cost volume into a probability volume using a 3D convolutional network [S1, unofficial].
    *   Sample depth planes: 64 planes for the coarse stage and 8 planes for the fine stage [S1, unofficial].
    *   Produce a depth prediction $D_t$ [S1, unofficial].
4.  **Gaussian Parameter Prediction (Pixel-aligned branch):**
    *   Pool the warped features $f_{t,i}$ into a target feature map $f_t$ [S1, unofficial].
    *   Pass $f_t$ through a 2D convolutional network and an MLP to predict scale $s$, rotation $q$, opacity $\alpha$, and color $c$ [S1, unofficial].
5.  **Position Assembly:**
    *   Unproject the predicted depth $D_t$ using the camera intrinsics $K$ to obtain the 3D position $p$ [S1, unofficial].
    *   Combine $s, q, p, \alpha, c$ to form the final attribute map $O_t$ [S1, unofficial].
6.  **Rendering and Reconstruction:**
    *   **Novel View Synthesis:** Render the scene using perspective-accurate 2D Gaussian Splatting rasterization [S1, unofficial].
    *   **Mesh Extraction:** Render mean depth maps $D_i$ for all training views $P_i$ using the inferred splat maps $O_i = \Phi(P_i | I_1, \dots, I_N)$ [S1, unofficial]. Perform TSDF fusion on these depth maps with a voxel size of 1.5 to extract the explicit 3D triangle mesh $S$ [S1, unofficial].

**Loss Functions and Training:**
The model is trained with a weighted sum of losses. The specific weights are:
*   $\lambda_s = 0.1$ (scale loss)
*   $\lambda_p = 0.05$ (position/depth loss)
*   $\lambda_\alpha = 0.05$ (opacity loss)
*   $\lambda_\beta = 0.05$ (rotation loss)
*   $\lambda_\gamma = 0.05$ (color loss)
*   $\lambda_1 = 0.5$ (L1 image loss)
*   $\lambda_2 = 1.0$ (SSIM image loss) [S1, unofficial]

The depth loss $L_{depth}$ guides the mean 2DGS splatted depth with ground truth depth supervision, which is critical for reducing chamfer distance [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class SparSplatModel(nn.Module):
    def __init__(self):
        super().__init__()
        # Feature encoders
        self.dinov2_encoder = get_dinov2_features() # Monocular semantic features
        self.mast3r_encoder = get_mast3r_features() # Pairwise dense correspondences
        self.fpn = FPN() # Predicts 2D feature map f_i
        
        # deepMVS branch
        self.cost_volume_net = CostVolumeNetwork()
        self.depth_head = DepthHead(coarse_planes=64, fine_planes=8)
        
        # Pixel-aligned prediction branch
        self.feature_pooler = FeaturePooler()
        self.gaussian_decoder = GaussianDecoder() # 2D Conv + MLP
        
    def forward(self, images: torch.Tensor, poses: torch.Tensor, intrinsics: torch.Tensor, target_pose: torch.Tensor):
        """
        images: [N, 3, H, W]
        poses: [N, 4, 4]
        intrinsics: [3, 3]
        target_pose: [4, 4]
        Returns: O_t (H, W, 13)
        """
        N = images.shape[0]
        
        # 1. Feature Encoding
        # Extract DINOv2 features for each image
        dinov2_feats = [self.dinov2_encoder(img) for img in images]
        
        # Extract MASt3R pairwise features
        mast3r_feats = self.mast3r_encoder(images) # Returns pairwise correspondences
        
        # FPN feature maps
        fpn_feats = [self.fpn(img) for img in images]
        
        # 2. Warping
        # Warp features to target view using relative poses
        warped_feats = []
        for i in range(N):
            rel_pose = target_pose @ poses[i].inverse()
            f_t_i = warp_homography(fpn_feats[i], rel_pose)
            warped_feats.append(f_t_i)
            
        # 3. Depth Estimation (deepMVS)
        # Merge warped features into cost volume
        cost_volume = merge_cost_volume(warped_feats)
        
        # Process into probability volume
        prob_volume = self.cost_volume_net(cost_volume)
        
        # Predict depth
        depth_map = self.depth_head(prob_volume) # Output: [1, H, W]
        
        # 4. Gaussian Parameter Prediction
        # Pool warped features into target feature map
        target_feat = self.feature_pooler(warped_feats)
        
        # Predict s, q, alpha, c
        gaussian_params = self.gaussian_decoder(target_feat) # Output: [1, H, W, 9] (s:2, q:4, alpha:1, c:3)
        
        # 5. Position Assembly
        # Unproject depth to 3D position
        # Note: This requires broadcasting intrinsics and depth
        positions = unproject_depth(depth_map, intrinsics) # Output: [3, H, W]
        
        # Combine all parameters
        # s: 2, q: 4, p: 3, alpha: 1, c: 3 -> Total 13
        O_t = torch.cat([
            gaussian_params[:, :, :, :2],   # s
            gaussian_params[:, :, :, 2:6],  # q
            positions,                      # p
            gaussian_params[:, :, :, 6:7],  # alpha
            gaussian_params[:, :, :, 7:]    # c
        ], dim=-1)
        
        return O_t

    def extract_mesh(self, images, poses, intrinsics):
        """
        Extracts 3D mesh from training views using TSDF fusion.
        """
        depth_maps = []
        for i in range(len(images)):
            # Infer splat map for training view i
            O_i = self.forward(images, poses, intrinsics, poses[i])
            # Render depth map from O_i
            D_i = render_depth(O_i, poses[i], intrinsics)
            depth_maps.append(D_i)
            
        # TSDF Fusion
        mesh = tsdf_fusion(depth_maps, voxel_size=1.5)
        return mesh
```

## Operating characteristics
*   **Time Complexity:** Inference is extremely fast, approximately 0.8–0.88 seconds on an RTX A6000 GPU for a single scene [S1, unofficial]. This is nearly two orders of magnitude faster than implicit generalizable SOTA methods like UfoRecon (66s), VolRecon (31s), ReTR (37s), and GeoTransfer (32s) [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On the DTU test set, SparSplat achieves a PSNR of 28.33, SSIM of 0.938, and LPIPS of 0.073 [S1, unofficial]. For 3D reconstruction (Chamfer Distance), it achieves a mean CD of 1.04 [S1, unofficial].
*   **Strengths:**
    *   Fast, feed-forward inference without per-scene optimization.
    *   Superior 3D reconstruction quality compared to prior generalizable GS methods (like MVSGaussian) due to 2DGS consistency.
    *   Strong generalization on unseen scenes (DTU, BlendedMVS, Tanks and Temples) [S1, unofficial].
    *   Dual capability for both high-quality novel view synthesis and explicit mesh extraction [S1, unofficial].
*   **Limitations:**
    *   Relies on external foundation models (DINOv2, MASt3R) which may introduce latency or dependency issues not present in self-contained networks.
    *   Performance is sensitive to the quality of the depth loss; without $L_{depth}$, reconstruction quality degrades significantly (Chamfer distance increases from 1.04 to 4.37) [S1, unofficial].
    *   Using 3DGS instead of 2DGS leads to depth inconsistencies across views, causing TSDF fusion to produce warped or disconnected surfaces [S1, unofficial].

## Background
SparSplat was proposed by Shubhendu Jena et al. in 2025 [S1, unofficial]. It builds upon the MVSGaussian framework [48] but replaces 3D Gaussian primitives with 2D Gaussian Splatting (2DGS) [28] to improve geometric consistency. The method integrates features from foundation models DINOv2 [59] and MASt3R [41] to enhance multi-view stereo accuracy. It addresses the limitation of prior generalizable GS methods which were optimized for novel view synthesis and failed to produce coherent meshes via TSDF fusion [S1, unofficial]. The work demonstrates that 2DGS is better suited for feed-forward surface reconstruction than 3DGS in sparse-view settings.

## Sources
[S1] SparSplat: Fast Multi-View Reconstruction with Generalizable 2D Gaussian Splatting, https://doi.org/10.48550/arXiv.2505.02175, fetched 2026-10-26 (unofficial)
