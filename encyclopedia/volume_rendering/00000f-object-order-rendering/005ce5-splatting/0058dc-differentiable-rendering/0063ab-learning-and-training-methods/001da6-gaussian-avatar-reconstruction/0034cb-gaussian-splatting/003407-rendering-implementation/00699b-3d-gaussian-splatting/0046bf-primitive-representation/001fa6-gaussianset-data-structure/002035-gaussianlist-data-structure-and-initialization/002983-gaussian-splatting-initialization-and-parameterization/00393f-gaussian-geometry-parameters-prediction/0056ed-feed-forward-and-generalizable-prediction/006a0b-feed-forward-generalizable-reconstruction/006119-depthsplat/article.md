# DepthSplat


## Overview
DepthSplat is a feed-forward method for 3D Gaussian splatting reconstruction that integrates pre-trained monocular depth features into a multi-view cost volume framework [S1]. It solves the problem of robust depth estimation in challenging scenes (occlusions, low-texture, non-Lambertian surfaces) where standard multi-view stereo fails, and scale ambiguities where monocular estimation fails [S1]. Unlike TranSplat, which refines coarse depth with monocular features (prone to error propagation), DepthSplat combines cost volumes and monocular features early in the pipeline [S1]. Unlike GS-LRM, it relies on explicit feature matching rather than pure data-driven priors, reducing training compute requirements significantly [S1].

## How it works
**Inputs:** $N$ input images $\{I_i\}_{i=1}^N$ with resolution $H \times W$, and corresponding camera projection matrices $\{P_i\}_{i=1}^N$ (derived from intrinsic and extrinsic parameters) [S1].

**Outputs:** A set of 3D Gaussians $\mathcal{G} = \{\mu_j, \alpha_j, \Sigma_j, c_j\}$ for each pixel $j$ in the reference views, where $\mu_j$ is the mean (3D position), $\alpha_j$ is opacity, $\Sigma_j$ is covariance, and $c_j$ is color [S1].

**Data Structures:**
*   **Monocular Features:** Extracted per-view using a Vision Transformer (ViT). Dimensions: $N \times C \times \frac{H}{s} \times \frac{W}{s}$, where $C$ is feature dimension and $s$ is downsampling factor [S1].
*   **Cost Volume:** For each view $i$, a 3D tensor $C_i \in \mathbb{R}^{\frac{H}{s} \times \frac{W}{s} \times D}$, where $D$ is the number of depth candidates [S1].
*   **Depth Map:** Predicted per-view dense depth $D_i \in \mathbb{R}^{H \times W}$ [S1].

**Parameters:**
*   **Downsampling factor ($s$):** Controlled by stride-2 $3 \times 3$ convolutions. $s=4$ with two convolutions; $s=8$ with three [S1].
*   **Depth candidates ($D$):** Uniformly sampled from near/far depth ranges [S1].
*   **Multi-view Transformer:** Contains six stacked self- and cross-attention layers [S1].
*   **Neighbor Selection:** For $N > 2$, top-2 nearest neighboring views are selected based on camera distance to the reference view [S1].

**Steps:**
1.  **Monocular Feature Extraction:** For each input image $I_i$, extract features $F_i$ using a pre-trained monocular depth model (ViT-based) [S1].
2.  **Cost Volume Construction (Multi-View Branch):**
    a. For each reference view $i$, uniformly sample $D$ depth candidates $\{d_m\}_{m=1}^D$ from the near/far ranges [S1].
    b. For each neighbor view $j$ (all views if $N \le 2$; top-2 nearest if $N > 2$), warp the features $F_j$ to view $i$ using $P_i$ and each depth candidate $d_m$ to obtain warped features $F_{j \to i}^m$ [S1].
    c. Compute feature correlations using the dot-product between $F_i$ and $F_{j \to i}^m$ [S1].
    d. If $N > 2$, average the correlation values from the two selected neighbor views [S1].
    e. Stack correlations for all $d_m$ to form cost volume $C_i \in \mathbb{R}^{\frac{H}{s} \times \frac{W}{s} \times D}$ [S1].
3.  **Feature Concatenation:** Concatenate the multi-view cost volumes $C_i$ with the extracted monocular features $F_i$ [S1].
4.  **Depth Regression:** Pass the concatenated features through a 2D U-Net architecture followed by a softmax layer to predict dense per-pixel depth $D_i$ [S1].
5.  **Gaussian Center Unprojection:** Unproject the predicted depth maps $D_i$ to 3D space using camera parameters $P_i$ to obtain Gaussian centers $\mu_j$ [S1].
6.  **Remaining Parameter Prediction:** Predict opacity $\alpha_j$, covariance $\Sigma_j$, and color $c_j$ using an additional lightweight network/head operating in parallel with the depth regression [S1].
7.  **Rendering:** Render novel views using the splatting operation with the predicted 3D Gaussians [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DepthSplat:
    def __init__(self, num_views, num_depth_candidates, downsampling_factor=4, 
                 feature_dim=256, neighbor_count=2):
        """
        Initialize DepthSplat components.
        :param num_views: N, number of input views
        :param num_depth_candidates: D, number of depth samples
        :param downsampling_factor: s, spatial downsampling factor
        :param feature_dim: C, feature dimension
        :param neighbor_count: K, number of neighbors for cost volume (2 for N>2)
        """
        self.num_views = num_views
        self.num_depth_candidates = num_depth_candidates
        self.downsampling_factor = downsampling_factor
        self.feature_dim = feature_dim
        self.neighbor_count = neighbor_count
        
        # Pre-trained monocular depth encoder (ViT)
        # Note: In implementation, this is a fixed weight network
        self.monocular_encoder = MonocularDepthEncoder()
        
        # Multi-view Transformer for feature alignment
        self.multi_view_transformer = MultiViewTransformer(
            num_layers=6, 
            feature_dim=feature_dim
        )
        
        # 2D U-Net for depth regression
        self.depth_decoder = DepthUNet(
            in_channels=feature_dim + feature_dim, # Concatenated cost vol + mono features
            out_channels=num_depth_candidates
        )
        
        # Lightweight head for Gaussian parameters
        self.gaussian_head = GaussianParameterHead(
            in_channels=feature_dim,
            out_channels=4 # alpha, scale(3), color(3) -> simplified for pseudocode
        )

    def forward(self, images, projection_matrices):
        """
        :param images: Tensor of shape (N, 3, H, W)
        :param projection_matrices: Tensor of shape (N, 3, 4)
        :return: List of Gaussian sets for each view
        """
        N, _, H, W = images.shape
        H_ds, W_ds = H // self.downsampling_factor, W // self.downsampling_factor
        
        # Step 1: Monocular Feature Extraction
        # Extract features from pre-trained model
        mono_features = self.monocular_encoder(images) 
        # Shape: (N, C, H_ds, W_ds)
        
        # Step 2: Cost Volume Construction
        cost_volumes = []
        
        # Determine neighbors for each view
        # For simplicity, assume projection_matrices allow distance calculation
        # In practice, compute pairwise camera distances
        neighbors = self._select_neighbors(projection_matrices)
        
        for i in range(N):
            # Reference view features
            F_i = mono_features[i] # (C, H_ds, W_ds)
            
            # Select neighbor indices
            neighbor_indices = neighbors[i]
            
            # Initialize cost volume accumulator
            # Shape: (H_ds, W_ds, D)
            C_i = torch.zeros((H_ds, W_ds, self.num_depth_candidates), device=images.device)
            
            # Sample depth candidates uniformly
            # Assuming near/far ranges are known or estimated
            depth_candidates = self._sample_depth_candidates(self.num_depth_candidates)
            
            for j in neighbor_indices:
                F_j = mono_features[j] # (C, H_ds, W_ds)
                
                # Warp F_j to view i using P_i and depth candidates
                # F_{j->i}^m shape: (C, H_ds, W_ds) for each m
                warped_features = self._warp_features(F_j, projection_matrices[i], depth_candidates)
                
                # Compute dot-product correlation
                # F_i: (C, H_ds, W_ds) -> (C, H_ds*W_ds)
                # warped: (D, C, H_ds, W_ds) -> (D, C, H_ds*W_ds)
                F_i_flat = F_i.view(F_i.size(0), -1) # (C, H_ds*W_ds)
                warped_flat = warped_features.view(warped_features.size(0), warped_features.size(1), -1) # (D, C, H_ds*W_ds)
                
                # Correlation: (D, H_ds*W_ds)
                correlations = torch.bmm(warped_flat, F_i_flat.unsqueeze(-1)).squeeze(-1)
                
                # Reshape to (H_ds, W_ds, D)
                correlations = correlations.permute(0, 2, 1).contiguous()
                
                # Accumulate (average if multiple neighbors)
                C_i += correlations
            
            # Average if multiple neighbors were used
            if len(neighbor_indices) > 1:
                C_i = C_i / len(neighbor_indices)
                
            cost_volumes.append(C_i)
            
        # Step 3: Feature Concatenation
        # Concatenate cost volumes and monocular features
        # Cost volumes are (N, H_ds, W_ds, D)
        # Monocular features are (N, C, H_ds, W_ds)
        # Need to align dimensions for U-Net input
        cost_volumes_tensor = torch.stack(cost_volumes, dim=0) # (N, H_ds, W_ds, D)
        cost_volumes_tensor = cost_volumes_tensor.permute(0, 3, 1, 2) # (N, D, H_ds, W_ds)
        
        # Expand mono_features to match batch
        mono_features_tensor = mono_features # (N, C, H_ds, W_ds)
        
        # Concatenate along channel dimension
        combined_features = torch.cat([cost_volumes_tensor, mono_features_tensor], dim=1)
        # Shape: (N, D + C, H_ds, W_ds)
        
        # Step 4: Depth Regression
        # 2D U-Net + Softmax
        depth_logits = self.depth_decoder(combined_features)
        # Shape: (N, D, H_ds, W_ds)
        
        # Softmax over depth dimension D
        depth_probs = F.softmax(depth_logits, dim=1)
        
        # Weighted sum to get depth map
        # depth_probs: (N, D, H_ds, W_ds)
        # depth_candidates: (D,)
        depth_map_ds = torch.sum(depth_probs * depth_candidates.view(1, -1, 1, 1), dim=1)
        # Shape: (N, H_ds, W_ds)
        
        # Upsample depth map to original resolution
        depth_map = F.interpolate(depth_map_ds.unsqueeze(1), size=(H, W), mode='bilinear', align_corners=False).squeeze(1)
        # Shape: (N, H, W)
        
        # Step 5: Gaussian Center Unprojection
        # Unproject depth map using projection matrices
        gaussian_centers = []
        for i in range(N):
            # Unproject pixel coordinates with depth
            # This involves back-projecting (x, y, 1) * depth using P_i^{-1}
            centers = self._unproject_depth(depth_map[i], projection_matrices[i], H, W)
            gaussian_centers.append(centers)
            
        # Step 6: Remaining Parameter Prediction
        # Use mono_features (downsampled) to predict alpha, sigma, color
        # Note: The source says "parallel" head. We use the same mono features or combined features.
        # Assuming head takes mono_features or combined features.
        # Let's use mono_features for simplicity as per "additional head" description.
        gaussian_params = self.gaussian_head(mono_features_tensor)
        # Shape: (N, C_params, H_ds, W_ds)
        
        # Step 7: Render (Optional for reconstruction, but part of method)
        # Render novel views using splatting
        # This step is typically used for training loss calculation
        # rendered_views = self._splat_render(gaussian_centers, gaussian_params, novel_view_matrices)
        
        return gaussian_centers, gaussian_params

    def _select_neighbors(self, projection_matrices):
        """Select top-2 nearest neighbors based on camera distance."""
        # Compute pairwise distances between camera centers
        # Camera centers can be extracted from projection matrices
        # For pseudocode, assume a helper function
        N = projection_matrices.shape[0]
        neighbors = []
        for i in range(N):
            # Placeholder for distance calculation logic
            # dists = compute_distances(projection_matrices[i], projection_matrices)
            # sorted_indices = argsort(dists)
            # neighbors.append(sorted_indices[1:self.neighbor_count+1]) # Exclude self
            # Simplified: return all if N <= 2, else top 2
            if N <= 2:
                neighbors.append([j for j in range(N) if j != i])
            else:
                # Mock selection
                neighbors.append([1, 2]) 
        return neighbors

    def _sample_depth_candidates(self, num_candidates):
        """Uniformly sample depth candidates from near/far ranges."""
        # Placeholder for depth range estimation
        # In practice, this might be global or per-scene
        near, far = 0.1, 100.0
        return torch.linspace(near, far, num_candidates)

    def _warp_features(self, features, projection_matrix, depth_candidates):
        """Warp features from source view to reference view using depth."""
        # Implement plane-sweep stereo warping
        # features: (C, H_ds, W_ds)
        # projection_matrix: (3, 4)
        # depth_candidates: (D,)
        # Returns: (D, C, H_ds, W_ds)
        pass

    def _unproject_depth(self, depth_map, projection_matrix, H, W):
        """Unproject depth map to 3D points."""
        # depth_map: (H, W)
        # projection_matrix: (3, 4)
        # Returns: (H, W, 3)
        pass

class MonocularDepthEncoder(nn.Module):
    def __init__(self):
        super().__init__()
        # Pre-trained ViT backbone
        pass
    def forward(self, x):
        return x # Placeholder

class MultiViewTransformer(nn.Module):
    def __init__(self, num_layers, feature_dim):
        super().__init__()
        self.layers = nn.ModuleList([
            nn.TransformerEncoderLayer(d_model=feature_dim, nhead=8) 
            for _ in range(num_layers)
        ])
    def forward(self, x):
        return x # Placeholder

class DepthUNet(nn.Module):
    def __init__(self, in_channels, out_channels):
        super().__init__()
        # 2D U-Net architecture
        pass
    def forward(self, x):
        return x # Placeholder

class GaussianParameterHead(nn.Module):
    def __init__(self, in_channels, out_channels):
        super().__init__()
        # Lightweight network head
        pass
    def forward(self, x):
        return x # Placeholder
```

## Operating characteristics
*   **Time Complexity:** The method is feed-forward, offering fast reconstruction. Reconstruction speed is 0.6 seconds for 12 input views at 512x960 resolution [S1].
*   **Memory Complexity:** Depends on the number of depth candidates $D$ and the cost volume size $\frac{H}{s} \times \frac{W}{s} \times D$. The multi-view transformer scales with the number of views, but neighbor selection limits this to top-2 neighbors for $N > 2$ [S1].
*   **Training Compute:** Significantly cheaper than data-heavy methods like GS-LRM. Trained in 2 days with 4 GPUs, compared to 64 A100s for GS-LRM [S1].
*   **Strengths:** Robust to occlusions, low-textured areas, and non-Lambertian surfaces by augmenting multi-view matching with monocular features [S1]. Maintains multi-view consistency while leveraging monocular priors [S1].
*   **Limitations:** Multi-view depth estimation still fails in extreme occlusions or textureless regions without monocular augmentation [S1]. Monocular features alone suffer from scale ambiguity, which is mitigated but not eliminated by the multi-view constraint [S1].
*   **When to Use:** Large-scale scene reconstruction from multiple views where robustness to challenging textures is required, and computational resources for training are limited [S1].
*   **When Not to Use:** Single-image reconstruction (monocular depth alone is insufficient) or scenarios where per-scene optimization is acceptable and higher fidelity is needed [S1].

## Background
DepthSplat was introduced to connect Gaussian splatting and depth estimation, studying their interactions [S1]. It addresses the limitations of traditional multi-view stereo (failure in occlusions/low-texture) and monocular depth estimation (scale ambiguity) [S1]. The method was trained on TartanAir, ScanNet, RealEstate10K, and DL3DV datasets [S1]. It demonstrates that Gaussian splatting can serve as an unsupervised pre-training objective for learning powerful depth models from large-scale multi-view posed datasets [S1].

## Sources
[S1] DepthSplat: Connecting Gaussian Splatting and Depth, https://doi.org/10.1109/cvpr52734.2025.01534, fetched 2026-10-26
